# FastPIR

FastPIR is a computational private information retrieval (CPIR) scheme that makes tradeoffs to improve Addra's performance, which requires low-latency responses from the server.
Although it was created for Addra, it can be used in place of other CPIR implementations like XPIR or SealPIR.

The main problem that all CPIR schemes attempt to tackle is to allow a client to retrieve information at some index from a server who owns some indexed database, without
allowing the server to obtain the index that the client is querying for. 

CPIR schemes generally consist of query, repsonse, and decode steps, and FastPIR is no different.

## Setup

### FastPIR Params

Before we can execute the FastPIR protocol though, we need to do some setup, especially when it comes to encryption. FastPIR heavily relies on the properties of the 
Brakerski/Fan-Vercauteren (BFV) homomorphic encryption scheme to achieve metadata privacy. More details about how FastPIR takes advantage of these properties
will come later, although a deeper dive into the inner workings behind BFV encryption is out of scope. 
Implementation-wise, FastPIR uses [Microsoft's Simple Encryption Arithmetic Library's (SEAL)](https://github.com/microsoft/SEAL) implementation of the BFV scheme.

Let's start by looking at `FastPIRParams`, which will be used to construct a FastPIR `Client` instance.

```cpp
FastPIRParams::FastPIRParams(size_t num_obj, size_t obj_size);
```

The constructor takes two parameters. In the context of Addra, `num_obj` is the number of mailboxes/users/rows in our database, and `obj_size` is the size of the message
stored in each mailbox, in terms of bytes.

Inside the constructor, we first set some parameters for SEAL's encryption scheme:

```cpp
seal_params = seal::EncryptionParameters(seal::scheme_type::BFV); // seal::EncryptionParameters
seal_params.set_poly_modulus_degree(POLY_MODULUS_DEGREE);
seal_params.set_coeff_modulus(COEFF_MOD_ARR);
seal_params.set_plain_modulus(PLAIN_MODULUS);
```

We first choose BFV encryption as our scheme. The other three variables can be tuned depending on the context.

`POLY_MODULUS_DEGREE`, also referred to as `N` in the Addra paper, determines the size of the ciphertext and the range of possible operations that can be performed
on the ciphertext. 

A ciphertext is just an encrypted plaintext. A plaintext is some polynomial representation of data; the specifics aren't necessary. All that you 
need to know is that a `seal::Plaintext` can be converted into `uint64_t` types, which are just 8 bytes. For the purposes of this article, `seal::Plaintext` is just
the data in a different form.

A larger value for `POLY_MODULUS_DEGREE` means larger ciphertexts and more complex operations, but it also decreases the speed of operations that can
be performed on the ciphertexts. `POLY_MODULUS_DEGREE` must be a power of 2 and defaults to `4096`; any lower and the range of operations is severely restricted. `4096`
is considered relatively small.

`COEFF_MOD_ARR` is an array of two large, distinct prime numbers. The size of the product of these two prime numbers determines the noise budget of BFV encryption.
At a high level, BFV encryption introduces some amount of noise when plaintext is converted into a ciphertext. As operations are performed on the ciphertext, the noise
budget is consumed. If the noise budget is exceeded, then decrypting the ciphertext will not return the original plaintext. The larger the product, the larger the noise budget.

`PLAIN_MODULUS` determines the size of the plaintext and the amount of noise plaintext operations will consume. A smaller plaintext modulus means less noise consumption.

Next, we find the number of ciphertexts in our query: 

```cpp
num_query_ciphertext = ceil(num_obj / (double)(POLY_MODULUS_DEGREE / 2));
```

`ceil(num_obj / POLY_MODULUS_DEGREE)` makes sense; `POLY_MODULUS_DEGREE` is the size of a ciphertext, so the number of ciphertexts we need to generate
is just enough ciphertexts to be able to cover all the mailboxes. So, if we have `4096` mailboxes and a `POLY_MODULUS_DEGREE` of `4096` as well,
we can query each row with a single ciphertext. If we had `8192`, we would need two. However, why do we divide `POLY_MODULUS_DEGREE` by `2`? This is actually an optimization,
and will be explained later when we explore the query step more.

Then, we find the number of columns for each mailbox, or the number of columns needed to store a message:

```cpp
num_columns_per_obj = 2 * (ceil(((obj_size / 2) * 8) / (float)(PLAIN_BIT)));
num_columns_per_obj += num_columns_per_obj % 2;
```

`PLAIN_BIT` is the number of bits in a plaintext; by default it is set to `18`. `PLAIN_MODULUS` must be greater than `2^PLAIN_BIT`, so it influences the size of `PLAIN_BIT`. 
The computation is a little confusing to me; my best guess is that we are getting the number of plaintexts or columns that we need to represent a message in a mailbox.
I think the `obj_size / 2 * 8` is converting the `obj_size` in bytes to bits, although I'm not sure why we divide by `2` and then multiply the result by `2`. The second
line seems to ensure that our `num_columns_per_obj` is even.

Then, we calculate the number of rows in our **encoded** database (this is not the same as the number of mailboxes!):

```cpp
db_rows = ceil(num_obj / (double)POLY_MODULUS_DEGREE) * num_columns_per_obj;
```

It is not too clear to me how this is calculated either, but it is related the forementioned query optimization.

### Client

Finally, now that all the FastPIR parameters have been calculated, we can see how the FastPIR client uses these parameters:

```cpp
Client::Client(FastPIRParams params)
{
    this->num_obj = params.get_num_obj();
    this->obj_size = params.get_obj_size();

    N = params.get_poly_modulus_degree();
    num_columns_per_obj = params.get_num_columns_per_obj();
    plain_bit_count = params.get_plain_modulus_size();
    num_query_ciphertext = params.get_num_query_ciphertext();

    context = seal::SEALContext::Create(params.get_seal_params());
    keygen = new seal::KeyGenerator(context);
    secret_key = keygen->secret_key();
    encryptor = new seal::Encryptor(context, secret_key);
    decryptor = new seal::Decryptor(context, secret_key);
    batch_encoder = new seal::BatchEncoder(context);

    std::vector<int> steps;
    for (int i = 1; i <= (num_columns_per_obj / 2); i *= 2)
    {
        steps.push_back(-i);
    }
    gal_keys = keygen->galois_keys_local(steps);

    return;
}
```

Almost all of it is just gathering the parameters set by `FastPIRParams` and configuring our SEAL library classes. 
`seal::SEALContext context` just checks the validity of the SEAL params we set earlier, and is used to configure other SEAL classes.
`seal::KeyGenerator keygen` allows us to generate a secret key for our client, which will be used for encryption/decryption.
The next line just generates and stores the secret key, and an `Encryptor` and `Decryptor` are constructed to encrypt/decrypt based off of this key.
The `seal::BatchEncoder` allows for the plaintexts to perform as `2 x N/2` matrices, like the ones mentioned in the Addra paper. This property allows for 
row/column rotations, and dramatically increases performance on these plaintexts.
`GaloisKeys` are required for rotation operations, which is necessary for optimizing the answer portion of FastPIR.

Now, we finally have our client set up!

### Server

Some setup has to be done on the server as well. The constructor also takes in the same `FastPIRParams` object:

```cpp
Server::Server(FastPIRParams params) {
  context = seal::SEALContext::Create(params.get_seal_params());
  N = params.get_poly_modulus_degree();
  plain_bit_count = params.get_plain_modulus_size();

  evaluator = new seal::Evaluator(context);
  batch_encoder = new seal::BatchEncoder(context);

  this->num_obj = params.get_num_obj();
  this->obj_size = params.get_obj_size();

  num_query_ciphertext = params.get_num_query_ciphertext();
  num_columns_per_obj = params.get_num_columns_per_obj();
  db_rows = params.get_db_rows();
  db_preprocessed = false;
}
```

Most of the constructor is simply just taking in the FastPIR params, since the server doesn't need to do too much encrypting. The `seal::Evaluator` is what allows
us to performs computations on ciphertexts.

## Query

Now, we can look at how our client can query for some index without the server knowing the specific index. A query is just a list of ciphertexts that encode our
query for the index. In code, this is just a `std::vector<seal::Ciphertext>`. The code to generate a query looks like this:

```cpp
  size_t slot_count = batch_encoder->slot_count();
  size_t row_size = slot_count / 2;

  for (int i = 0; i < num_query_ciphertext; i++) {
    std::vector<uint64_t> pod_matrix(slot_count, 0ULL);
    if ((index / row_size) == i) {
      pod_matrix[index % row_size] = 1;
      pod_matrix[row_size + (index % row_size)] = 1;
    }
    batch_encoder->encode(pod_matrix, pt);
    encryptor->encrypt_symmetric(pt, query[i]);
  }
```

First, we obtain the `slot_count` and `row_size` from the `batch_encoder`. The `slot_count` is just equal to `N` or the `POLY_MODULUS_DEGREE`, which is just the size of our
ciphertext. Since the batch encoder treats a ciphertext as a `2 x N/2` matrix, `slot_count / 2` just gives us the size of the row, which is `N/2`.

Now, we generate `num_query_ciphertext` amount of ciphertexts for a query. At a high-level, we want to generate a 1-hot encoding for our query, which is just a all `0` vector 
with a `1` representing the index we want. Then, all the server has to do is multiply this 1-hot encoding against its encoded database, and the `0`s will cancel all the other 
data out and leave us with the data we're interested in. That is what the loop is doing; for each ciphertext, it initializes it to all `0`s, and then checks if the index we're
querying for exists within this ciphertext. If so, we set the index for this ciphertext equal to `1`. Then we just encode the data into a plaintext, and then encrypt
that plaintext into a ciphertext and store it in our query.

However, notice that we set two values in our ciphertext equal to `1`; doesn't that break our 1-hot encoding? Actually, this is the optimization mentioned before.
Notice how `num_query_ciphertext` is double the size needed to query all the rows in our database; this is actually because we can query two rows in our database at
the same time. Every ciphertext we're generating looks something like `[[0, 0, 0, 0], [0, 0, 0, 0]]`, due to the batch encoder. Now, we can treat this ciphertext like it's
querying two rows, where the first half of the ciphertext is querying the first row and the second is querying the second. So, if we have `[[0, 0, 1, 0], [0, 0, 1, 0]]`, we
can get two values from a row with one multiplication operation. Again, this makes the query size two times larger, but it reduces some expensive rotation operations that
we'll get to in the answer step.

In the context of Addra, this tradeoff is worth it, as it makes the initial dialing phase more expensive but allows for much cheaper answers, which is where we typically 
bottleneck.

In order to use this optimization though, our database has to be altered. The following code initializes and encodes a database to be compatible with this optimization:

```cpp
void Server::set_db(std::vector<std::vector<unsigned char>> db) {
  assert(db.size() == num_obj);
  std::vector<std::vector<uint64_t>> extended_db(db_rows);
  for (int i = 0; i < db_rows; i++) {
    extended_db[i] = std::vector<uint64_t>(N, 1ULL);
  }
  int row_size = N / 2;

  for (int i = 0; i < num_obj; i++) {
    std::vector<uint64_t> temp = encode(db[i]);

    int row = (i / row_size);
    int col = (i % row_size);
    for (int j = 0; j < num_columns_per_obj / 2; j++) {
      extended_db[row][col] = temp[j];
      extended_db[row][col + row_size] = temp[j + (num_columns_per_obj / 2)];
      row += num_query_ciphertext;
    }
  }
  encode_db(extended_db);
  return;
}
```

First, we encode each object so that it is in a form that can be encoded into plaintext. Then, we insert each encoded item into our extended database.
The code has similarities to the query generating code, in that we need to encode our data in a multidimensional manner.

The `encode_db` function is simpler:

```cpp
void Server::encode_db(std::vector<std::vector<uint64_t>> db) {
  encoded_db = std::vector<seal::Plaintext>(db.size());
  for (int i = 0; i < db.size(); i++) {
    batch_encoder->encode(db[i], encoded_db[i]);
  }
}
```
It just converts the extended database into plaintext.

## Answer

The answer is generated on the server from the query. The function looks like so:

```cpp
PIRResponse Server::get_response(uint32_t client_id, PIRQuery query) {
  if (query.size() != num_query_ciphertext) {
    std::cout << "query size doesn't match" << std::endl;
    exit(1);
  }
  seal::Ciphertext result;
  preprocess_query(query);
  if (!db_preprocessed) {
    preprocess_db();
  }

  seal::GaloisKeys gal_keys = client_galois_keys[client_id];
  seal::Ciphertext response =
      get_sum(query, gal_keys, 0, num_columns_per_obj / 2 - 1);
  return response;
}
```
The `preprocess_`-prefixed functions take a vector of plaintexts or ciphertexts and performs a number-theoretic transform (NTT) on them. Number-theoretic transforms are out of scope,
but essentially they allow for fast convolutions on large number sequences, which is useful for increasing the speed of multiplications.

The brunt of the work is done in `get_sum`:

```cpp
seal::Ciphertext Server::get_sum(std::vector<seal::Ciphertext> &query,
                                 seal::GaloisKeys &gal_keys, uint32_t start,
                                 uint32_t end) {
  seal::Ciphertext result;

  if (start != end) {
    int count = (end - start) + 1;
    int next_power_of_two = get_next_power_of_two(count);
    int mid = next_power_of_two / 2;
    seal::Ciphertext left_sum =
        get_sum(query, gal_keys, start, start + mid - 1);
    seal::Ciphertext right_sum = get_sum(query, gal_keys, start + mid, end);
    evaluator->rotate_rows_inplace(right_sum, -mid, gal_keys);
    evaluator->add_inplace(left_sum, right_sum);
    return left_sum;
  } else {

    seal::Ciphertext column_sum;
    seal::Ciphertext temp_ct;
    evaluator->multiply_plain(
        query[0], encoded_db[num_query_ciphertext * start], column_sum);

    for (int j = 1; j < num_query_ciphertext; j++) {
      evaluator->multiply_plain(
          query[j], encoded_db[num_query_ciphertext * start + j], temp_ct);
      evaluator->add_inplace(column_sum, temp_ct);
    }
    evaluator->transform_from_ntt_inplace(column_sum);
    return column_sum;
  }
}
```

Essentially, `get_sum` recursively calls itself on each half of the database. Once we're at a single database column (or 2), we multiply all of the query ciphertexts
with all of the rows in the column(s), extracting the data at these column(s). We accumulate the results, since accumulating `0`s won't affect the sum. Finally,
we convert the ciphertext from it's NTT form back into it's regular ciphertext form, and return it.

When `get_sum` gets results from its two halves, it combines the sums by rotating the ciphertexts. Without this step, we are wasting a lot of memory, since the majority
of results from `get_sum` will be primarily `0`s. By rotating, we can fill the empty space. For example, if we have `[0, 0, a, 0]` from column 1 and `[0, 0, b, 0]` from
column 2, there's no need to return both of them; why not combine them by rotating one and then summing them? So, we can rotate `[0, 0, b, 0] => [0, 0, 0, b]`, then 
combine that with the result from column 1 to obtain `[0, 0, a, b]`. By recursively combining, we can end up with a single ciphertext, instead of `num_query_ciphertext` amount
of ciphertexts. 

The decision to split based off of the `next_power_of_two` is important as well; BFV row rotations are fast for powers of two, since non-powers of two have to be composed of
powers of two. 

Once the two halves have been encoded and merged, we can return the ciphertext to the user.

## Decode

Finally, the client receives the ciphertext and can decode it. The code looks like this:

```cpp
std::vector<unsigned char> client::decode_response(PIRResponse response,
                                                   uint32_t index) {
  seal::plaintext pt;
  std::vector<uint64_t> decoded_response;
  size_t row_size = n / 2;
  decryptor->decrypt(response, pt);
  batch_encoder->decode(pt, decoded_response);

  decoded_response = rotate_plain(decoded_response, index % row_size);

  return decode(decoded_response);
}
```
First, we decrypt the ciphertext into a plaintext, then decode the plaintext into a list of `uint64_t`. However, due to our rotation operation, the response is now rotated.
We just need to rotate it back based off of our index, then decode the rightfully rotated response into a list of bytes.

## Recursion

Currently, the query size is of order `O(N)`, where `N` is the number of rows/mailboxes in the system. This is true for most CPIR schemes, although Kushilevitz, Ostrovsky,
and Stern have proposed a recursive scheme to decrease the size of the query. It does so by converting the database (initially 1-dimensional, ignoring the dimension for 
the messages) into `d` dimensions. So, if `d=2`, then our database of dimension `N x 1` will be converted to a database of `N^(1/2) x N^(1/2)`. Then, instead of generating a 
single 1-hot encoded query ciphertext (ignoring partitioning) for the index, we generate `d` query ciphertexts for each dimension. Clearly, the query size is now of order
`O(dN^(1/d))`.

Then, when the server receives these `d` query ciphertexts, it continues multiplying using each query ciphertext, and on each multiplication we isolate some dimension
of the database. For example, given `d=2`, we have two query ciphertexts, one for the rows and one for the columns. If we multiply each row by the column ciphertext, we
have extracted the columns that we are interested in. Then, when we multiply the columns by the row query ciphertext, we isolate the exact element that we're interested in.
This can be generalized to `d` dimensions. The answer step now requires more multiplications. Originally we only needed `O(N)` multiplications (we partition the rows based 
off of `N`, the size of the ciphertext), but now this costs `O(dN)`. 

This method makes a tradeoff between the size of the query and the amount of time consumed in the answer step. For `d < log(n)`, communication becomes sublinear, but larger
values of `d` makes communication superlinear.

## Sources
- [FastPIR paper](https://eprint.iacr.org/2021/044.pdf)
- [FastPIR source code](https://github.com/ishtiyaque/fastpir)
- [Recursion](https://eprint.iacr.org/2019/1483.pdf)


