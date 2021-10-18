# FastPIR

FastPIR is a computational private information retrieval (CPIR) scheme that makes tradeoffs to improve Addra's performance, which requires low-latency responses from the server.
Although it was created for Addra, it can be used in place of other CPIR implementations like XPIR or SealPIR.

The main problem that all CPIR schemes attempt to tackle is to allow a client to retrieve information at some index from a server who owns some indexed database, without
allowing the server to obtain the index that the client is querying for. 

CPIR schemes generally consist of query, repsonse, and decode steps, and FastPIR is no different.

## Setup

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

It is not too clear to me how this is calculated either, but I think it is related the forementioned query optimization.

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


