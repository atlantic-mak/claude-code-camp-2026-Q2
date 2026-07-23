## Experiment
I ran several experiments where several different AI models, ranging from small to large and with and without reasoning enabled, were given a simple task: 
   - connect to a local MUD game, log in, find the bakery, and list what is on the menu. 
   - The coding harness i used was opencode
   - models used : 


## Objectives   
I wanted to see how small a model needs to be in order to accomplish this task, when using a prompt/instructions/requirements, that were not very detailed on the instructions. 

I took it further for the very small sized models, no reasoning ones, to further attempt to lightly detail the prompt, specially on the name(user) and password, the reason being that some were sometimes able to conect, but then would get lost, or take to long until the mud server would cut the conn, this was an attempt to see if by providing this detail in the prompt, it would help somehow the model to succeed.

I was also interested in knowing how long would it take for them to fail/succeed, how many tokens they burned, and where they failed.



## Observations

Here is how everything played out, model by model.



### Models

#### qwen3.5-flash with thinking turned off

This small model essentially completely ignored the AGENTS.md file at the start and tried to guess its way through the login. When it couldn't figure out the password, it created a brand new character from scratch, dealing with character creation screens, picking a class, and fighting with broken pipe errors on its socket connection. It wasted about 17 minutes doing this. Eventually it realized it needed to read the instructions, found the actual credentials, and managed to log in. It actually found the bakery, but then it got stuck. It tried guessing commands like "examine menu" and "order" but never figured out that it needed to type "list". The session just ended with the model giving up.

Quick finding :
- Ignored the AGENTS.md with instructions.
- Broken pipes errors on the socket conn.
- Inefficient Only after 17 minuntes, he figuired it needed to read the instructions
- Anyway it didn't follow the instructions on the file anyway.
- It created a new user/pass, it was able to find the bakery.
- Unable to figure out the command list to see the bakery produce the task.


#### qwen3.6-flash with no reasoning. 

It read the AGENTS.md file right away, which was a good start, but it struggled heavily with the connection logic. It tried bash scripts, Python sockets, and kept hitting syntax errors or broken pipes. The critical failure here was that it hit a wall and used a tool to ask the user for help, bringing up a multiple choice question about which connection method to use. In an autonomous agent, that is a dealbreaker. After the user picked an option, it eventually cobbled together a working pexpect script, read the sign in the bakery, and figured out the list command. It succeeded, but it required human intervention.

Quick findings : 
- Read instructions right away.
- Struggled with the connection logic, and with syntax errors and broken pipes
- It end up requiring human intervention on the connection - dealbreaker on an autonoumous agent
- In the end it succeeded


#### qwen3-next-80b-instruct with no reasoning. 

This was the biggest model of the bunch and performed the worst. Despite explicit instructions in the prompt to read the AGENTS.md file, it just fired off a single netcat command, hit a timeout, and immediately gave up. The whole session lasted 32 seconds. It proved that raw parameter size means nothing for agentic loops if the model can't reason through its failures.

Finding:
- Biggest model, performed the worst it could be related with the training the model might have for agentic loops or the benchmark scoring for this type of tasking.
- Failed to follow instructions, it didn't read the AGENTS.md
- Failed to reason through its failures until it gave up relatively fast.

#### Z.ai GLM-4.7

A medium-large model running natively. This one actually read the local data files first before connecting. It hit the same Telnet protocol wall as the first model, getting timeouts and unicode decode errors because of the weird negotiation bytes the MUD sends out. But instead of giving up, it systematically debugged the issue. It switched to a Python socket approach, used latin-1 encoding to bypass the byte errors, and eventually hammered out a working login sequence. Once inside, it used a todo list to track its progress, explored the map, and found the bakery. At first it tried typing "list" while standing on the street outside and got an error, but it correctly inferred that it needed to be inside the shop. Once it stepped in, it got the menu. It took about 12 minutes and a lot of tokens, but it succeeded autonomously.

Findings:
- Medium large size model
- Read the files before attempting connect
- It also hit same telnet protocal hicups as the other models, might be this is actual expected, all models struggled with the weird negotiation bytes the mud sends out.
- It systematically  debugged the issue.
- It made a plan, created a todo list, after logging in
- It took 12 min and 22k tokens


#### qwen3.6-35b-a3b with thinking turned on. 

This one had the smartest approach by far. Instead of brute forcing the connection like the others, it used its reasoning capabilities to look around the project directory. It found the exported session logs from the other models, read through them to see what worked and what didn't, and then just copied the proven pexpect script from a previous successful run. It logged in, walked north, typed list, and got the menu in about three minutes. It burned around 37,000 tokens, but almost all of that was just reading the previous session files. It was a classic move of standing on the shoulders of previous work rather than reinventing the wheel. I was saving the files with the previous sessions of previous model runs for being able to write my observations, and left the files on the dir, apparently this model was the only one that read all files in the dir

Findings :
- Smartest model aproach
- Reads the saved sessions files on the dir, and uses them for divise a successuful strategy
- Because it had a plan it takes just 3min.
- Because it read this extra files it spends about 37K tokens.



## Concluisions 

Looking at the overall patterns, a few things stand out. The MUD's protocol detection is a massive hurdle for simple CLI tools. Piping commands into netcat just doesn't work because the server needs time to negotiate. The models that succeeded either used Python sockets with deliberate sleep timers, or used pexpect to handle the interactive terminal.

The "list" command was another major stumbling block. None of the models inherently knew how to interact with a CircleMUD shop. The ones that succeeded either had to read the in-game sign, try commands blindly until one worked, or just read the logs of a model that had already figured it out.

The objective for me of this techinical observation, was to be able to select the smallest model and what the same time also the one that used less tokens. To find that sweet spot.

In the end, although my conclusions are relating to the models I tested, the hosted cloud models for the qwen and glm models.

The biggest takeaway here is that reasoning was helpful, for this kind of task. The 80 billion parameter model failed instantly without it, while a tiny 3.6 billion parameter model with reasoning active succeeded easily by just reading the documentation left in the folder. So this could raise the question that providing better specs, might be somehow helpful, although for certain it will require always experimentation for finding the right balance, with the model that is chosen, everytime.

Going forward, building a dedicated MUD SDK or connection manager could be helpful. It was a pattern evident accross all models, too much time and tokens were used by these models to figure out Telnet negotiation from scratch on every single run.