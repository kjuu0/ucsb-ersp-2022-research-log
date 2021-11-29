# Research Log

## Week 10 (11/29 - 12/05)

### Goals

- [ ] Do more research into SEAL
- [ ] Prepare the final presentation slides
- [ ] Last revision of our proposal

### Accomplishments

#### Monday (3 hours)

- Worked with Allison on presentation slides
  - Wrote slides on the protocol
- Worried about content pacing & time
- Set up meeting with Ishtiyaque for next steps on SEAL parallelization & grad student interview for tomorrow

## Week 9 (11/22 - 11/28)

### Goals

- [x] Familiarize self with SEAL
- [x] Define concrete goals for SEAL familiarization
- [x] Revise proposal

### Accomplishments

#### Wednesday (0.5 hours)

- Met with Allison to discuss some concrete goals for SEAL familiarization, as well as a rough deadline (next Monday)

#### Tuesday (2 hours)

- Added section on how components interact and how a direct phone call works

#### Monday (1.5 hours)

- Received feedback about our proposal from other students
  - From my reviewer specifically:
    - Define more terms (RPC, full duplex, etc.)
    - Generate more motivation for metadata privacy (like % of people who get their data stolen??)
    - Give more connection between components in the architecture
  - Aggregated feedback from other students
    - Define more terms
    - Give more importance/motivation for metadata privacy (specific examples)
    - Give a high level overview at the end as to how the components of the client & server connect (we can reference the figures here)
    - Explain more (at a high level?) how Addra works
    - Clear up the thesis? (I thought it was clear enough but ok)
    - Add a nicer title
- Divided up the work amongst teammates

## Week 8 (11/15 - 11/21)

### Goals

- [x] Add more detail (specifically implementation) to the proposal

### Accomplishments

### Friday (1 hour)

- Met with Ishtiyaque to discuss side SEAL parallelization project
- Loose guideline of what to expect and what the project is aout

### Wednesday (1 hour)

- Added diagram for server side architecture
- Defined server-side components and how they interat

### Tuesday (1 hour)

- Added diagram for client side architecture
- Defined more components and how they interacted

#### Monday (2 hours)

- Met with team and highlighted areas to improve
  - More detail on implementation

## Week 7 (11/08 - 11/14)

### Goals

- [x] Dive deeper into PortAudio, LPCNet, AWS EC2

### Accomplishments

#### Friday (2 hours)

- Met with team and worked on our proposal
  - Planned out revision strategies for part 1 & 2
  - Wrote our evaluation & implementation plan for part 3
- Created a [Kanban board to track things we need to do](https://github.com/kjuu0/cs110-proposal/projects/1)

#### Wednesday (1 hour)

- Reviewed and added to our [proposed solution](https://www.overleaf.com/project/618062838f6fa9d90c073fdb)

## Week 6 (11/01 - 11/07)

### Goals

- [x] Decide on a research problem
- [x] Do more research into the research problem

### Accomplishments

#### Friday (2 hours)

- Met with team to plan out a high-level architecture for our system utilizing Addra
  - [Diagram](https://drive.google.com/drive/u/1/folders/1HACM6sKpT_VdpdnxVF3nLHT7EG-uxq0M)
  - Looked into voice encoding libraries like LPCNet, audio I/O libraries like PortAudio, cloud solutions for the backend like AWS EC2

#### Wednesday (2 hours)

- Finished writing the intro to our [research proposal](https://www.overleaf.com/project/618062838f6fa9d90c073fdb)
- Met with Ishtiyaque and the professors and informed them about our decision

#### Tuesday (0.5 hours)

- Talked to team and chose a research problem between 1) GPU parallelizing FastPIR, 2) utilizing Addra in an end-to-end system, supporting actual voice calls, and 3) reworking Addra for asynchronous applications
  - Nico and I were heavily in favor for GPU, but Jerry did not like it much, Allison seemed impartial
    - Jerry was primarily worried about the time investment for learning GPUs, which I also was worried about it (but I think we could've learned it)
    - Nico and I liked the learning opportunity
  - In the end, we decided on option 2, since none of us were against it like the GPU project
    - Good opportunity to learn about voice encodings
    - Client-server architecture
    - Real application

#### Monday (1 hour)

- [Read and summarized research article on GPUs](https://docs.google.com/document/d/17aSD51Y-_eXkkHOTgMwNBWvz1hmFJgX7GAKLw4llidg/edit)

## Week 5 (10/25-10/31)

### Goals

- [x] Investigate research topics/questions

### Accomplishments

#### Tuesday (2 hours)

- Contributed/started the [literature search for our group](https://docs.google.com/document/d/17aSD51Y-_eXkkHOTgMwNBWvz1hmFJgX7GAKLw4llidg/edit)

#### Monday (1 hour)

- Brainstormed research problems/topics
  - High-level:
    - Preserve metadata privacy between two users in a system
    - Improve scalability of the system
    - Decrease latency of the system
    - Optimize resource usage of the system
    - Applying Addra to real-world use cases
  - Low-level:
    - Optimizing the Addra protocol to scale better through parallelism (CPU or GPU)
    - Decreasing general communications costs   

## Week 4 (10/18-10/24)

### Goals

- [x] Prepare teaching materials on FastPIR
- [x] Investigate recursion in CPIR schemes

### Accomplishments

#### Wednesday (0.5 hours)

- Finished scheduling a weekly meeting time with Ishtiyaque

#### Tuesday (2 hours)

- Did more investigation into recursion and FastPIR, details are in the same [notes](papers/fastpir.md)

## Week 3 (10/11-10/17)

### Goals

- [x] Start doing more specific research into assigned topics (FastPIR)
- [x] Look more into Addra/FastPIR source code

### Accomplishments

#### Sunday, 10/17 (3 hours)

- Looked more deeply into FastPIR and the FastPIR source code and took [notes](papers/fastpir.md)

#### Friday, 10/15 (1.5 hours)

- Talked with team about our next steps
- Started discussing how we could go about creating a real voice call

#### Wednesday, 10/13 (2 hours)

- Did more reading into BFV encryption & CPIR
- Gave the Addra paper some additional passes

## Week 2 (10/04-10/10)

### Goals

- [x] Read the how to read an article paper
- [x] Finish first pass of reading/understanding/taking notes the intro material that Ishtiyaque provided

### Accomplishments

#### Friday, 10/08 (0.5 hours)

- Met with the team and discussed which topics we should do more research into

#### Wednesday, 10/06 (1 hour)

- Attended research meeting & got more context on the possible projects

#### Tuesday, 10/05 (4 hours)

- Took notes on a brief first pass of [Addra](papers/addra.md)

#### Monday, 10/04 (1 hour)

- Finished reading the "How to Read a Paper" article

## Week 1 (09/27-10/03)

### Goals

- [x] Communicate with professors to get a finalized research meeting time
- [x] (Maybe) Attend the first research meeting
- [x] Set up research log
- [x] Complete Reflection 1: Identity and ERSP preliminary thoughts
- [x] Meet the rest of the team
- [x] Reflect on research logs

### Accomplishments

#### Sunday, 10/03 (1 hour)

- Communicated with Ishtiyaque (research mentor) to start setting up weekly meetings
  - Waiting on Ishtiyaque to provide his availability
- Set up a group page  

#### Thursday, 09/30 (1 hour)

- Attended group meeting session
- Set up meeting on Monday (10/04) with Chinmay

#### Wednesday, 09/29 (1 hour)

- Attended research meeting

#### Tuesday, 09/28 (2 hours)

- Set up research log. Chose to use Github Markdown because I'm a fan of the aesthetics/formatting and it makes linking other documents in the same repo easy.
- Finished Reflection 1.
- Finished reading & [reflecting on the research logs.](/LOG_REFLECTIONS.md)
- Finalized a meeting time for the team on Slack for Thursday 4-5 PM.
- I'm most excited about:
  - Meeting my team/professors
  - Stepping out of my comfort zone
  - Doing research on distributed systems
- I'm most nervous about:
  - Keeping up with my teammates
  - Balancing my schedule
