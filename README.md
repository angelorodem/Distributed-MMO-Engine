# Distributed-MMO-Engine (WIP)
Teorical example of a distributed Massively Multiplater Online Game (MMO)

Welcome to the MMO Architecture project! Here you will find the core concepts of how a distributed Massive Multiplayer Online game (networking and software-wise) works.

The first inspiration for this project is how a game like World of Warcraft or Lineage works, and how we can make a massive world function seamlessly to the player while being distributed and load-balanced behind the scenes.

The second is to teach people about different technologies and how they can be applied to multiplayer gaming, explain why some technical decisions were made and why.

The concepts shown in this article are going to be tool agnostic, however, In this repository, there is also an implementation of all the discussed concepts in C++ using the best open source technologies currently available, some of which are:

- **FlatBuffers** for client data serialization (sending and receiving player data)
  - Fast serialization and deserialization
  - Zero copy
  - Small memory footprint
  - Good community support
  - Can operate with gRPC replacing ProtoBuffers
- **ASIO C++ Networking Library** for performant UDP packet exchange
  - Good performance
  - Good community support
  - Good C++11 support
- **gRPC** w/ **FlatBuffers** for distributed execution of intensive code in the backend
  - Easy implementation
  - Good community support
  - Wide range of languages supported
- **MariaDB** (Open Source MySQL) for transactional, low flux, high complexity data
  - Good performance
  - Same structure as MySQL
  - Good community support
- **MongoDB** w/ **Redis** Caching for high flux game state storage of dynamic data
  - Redis is extremely fast
  - MongoDB is fast and flexible
  - Easy to scale
  - Good support

The client will also be implemented in Python to explain how Python and C++ can work together through bindings.

This project has a goal of being distributed and highly available, so it uses individual components for each core functionality (as shown in the diagram). Every single one is divided into its own repository, so each can be explained in more detail there. Every repository is listed below:

- [...]

All the components used to make the architecture for our game server are shown in the figure below:

![](RackMultipart20200531-4-14essvt_html_68a32e2a22d54499.png)

To explain this whole diagram I&#39;ll tell a story of a normal user playing World of Warcraft. The discussed topics are:

1. **Module overview?**
2. **The Journey Begins! 🙌**
  - Account Creation
  - Client Updates
  - Login
  - Pint of Cryptography
  - Realm Choosing
  - Character Choosing and Creation
3. **Now It just got Complex 😱**
  - Entering the World
  - Game State
  - Relevant data and Client Side data
  - Region of Interest
  - Loading dynamic objects (NPCs, Players, Mail Boxes, Anvils...)
4. **Interacting through the Realm 🌎**
  - Sending a message to a friend
  - Checking auctions
  - Check mail
  - Invite friends to Raid
5. **Now It just got Complex 😱 2: Revengeance**
  - Passing from a region to other while walking
  - Enter instance
6. **Finally some ACTION! 💪**
  - Aggro and simple AI
  - Attacking a simple monster
  - Attacking a scripted monster (with some kind of AI)
  - Being affected by a script (Boss spell that pushes players backwards with high speed)
7. **Players will be players 😈**
  - Authoritative server &amp; Dumb clients
  - Checking if a player is cheating (NoClip, Speed hack, etc...)
  - Protocol exploitation
8. **Unfortunately we are in the real world 😢**
  - Networking
  - Distributed server
  - Networking limitations (Tick rate)
  - Lag and packet losses
  - Client side predictions techniques
  - Cloud
  - Realm Distribution

1.
## The Journey Begins 🙌

Matthew also known as Matt (Cool guy though) receives a copy of the game client from a friend. The first thing that he does is open the game website and create an account filling the necessary account fields, checking all these boxes that say that he read something that he didn&#39;t.

When Matt clicks on accept, the data from the form is sent to the website backend, which has access to the **Account Database.** An SQL query creates a new entry in the database which will have at least a **Player Identifier (PI)** (Name, nick or email) and a **Player Secret (PS)** (password or a token) that will be used to log into the game.

Matt then proceeds to play the game. When he opens the client, the first step the client does is to **check for new updates**.

The process of checking for updates is quite simple: the client iterates through the game directory generating [**Hashes**] for every file, these hashes are then compared with remote file hashes (held by the **Client Patching Server** ), and if these don&#39;t match, the file needs to be **downloaded again**.

Downloading the newest file is the ~~laziest~~ simplest way of updating, although it&#39;s not the only approach. Some techniques like file [**Data patching**] can be applied as well, but these are very complex, although they can reduce the number of full downloads, reducing network traffic.

The client receives the new update data from a [**CDN**] and after being fully updated it allows Matt to insert data in the two login fields: Email ( **PI** ) and Password ( **PS** ). Matt fills both inputs and clicks on the button &quot; **Login**&quot;.

When the login procedure starts, the first things to happen is the [**Secure hashing**] of both inputs, hashing these inputs helps hiding the original contents of the fields, though only hashing does not protect against credential-stealing if a secure communication channel is not established because it will be vulnerable to [**Replay Attacks**].

Secure channels are made using a mixture of symmetric and asymmetric encryption. Asymmetric encryption is slow and can&#39;t be used alone to secure data, so symmetric encryption (which is blazing fast) is used to secure the data while asymmetric secures the keys.

Symmetric encryption uses only one key that is used to both encrypt and decrypt data. This key can&#39;t be transferred via the internet as it may be stolen through a [**Man in the Middle**] attack, hence it needs to be encrypted with a public asymmetric key (which can be known), allowing it to be securely transmitted to the server.

ToThe following diagram helps visualizing a secure channel communication:

![](RackMultipart20200531-4-14essvt_html_64243740fd56d1e8.png)

This cryptography content may seem to be a little daunting, but all of these steps are already implemented in libraries like OpenSSL, Crypto++, Botan, and cryptography (in Python), making the implementation of this flux very easy.

When the communication is established with the **Connection Manager (CM)** it forwards the credentials to the **Login Manager** , which then checks the database for a matching email and password pair (using the hashes) Assuming that Matt has inserted his credentials correctly and chose a realm, the Login manager will either put him in a queue, or proceed with the login procedure.

Assuming that there was no queue to log in, the **Login manager** will send the client a unique **Login Token** , which indicates that there was a successful login to a **Realm**. This Login token is a small encrypted token used to verify which user sent which packet and if it is correctly authenticated similar to [**JWT**].

With the Login token, the client will also receive the IP of a **Realm Connection Manager (RCM)** to connect, which will be the middleman for the client and **Realm specific servers** later.

The **RCM** receives from the **CM** the data which is encrypted in the Login token and stores it for checking incoming user packets. Later it will also receive the information to which **Region Server (RS)** the player will connect, informed by the **Objects Exchange Manager (OEM)**.

**OEM** is the server responsible for the player objects, it will also be responsible for the exchange of objects between servers (like players and NPCs), which is explained in detail later.

At this point the **RCM** knows that Matt has been successfully logged in, it now waits for the client to connect using the received IP and the Login Token.

The Client now connects with the **RCM** and asks for basic character information, **RCM** receives this request and forwards it to the **Realm Wide Actions Manager (RWAM)**.

**RWAM** is responsible for everything that is not location-based in the game (like character creation, mail sending, auctions, Party, Guilds...), so as it receives this request to show the available characters, it queries the **Realm Data (RD)** database for the information.

As is the first time Matt has played this game, the **RWAM** returns no character, making the client open a **New Character Screen** , Matt then proceeds to choose his faction, character preferences and (in a lack of creativity) names his new Human Warrior &quot;Matt&quot;.

All the &quot;filled&quot; data in the New Character Screen is then [**Serialized**] and fit on packet sent to **RWAM** , which upon receiving it, checks firstly if it&#39;s a real, valid and well-formed packet (and not an attack like a [**Malformed packet**] or a [**Buffer Overflow**]), then checks if the data itself is valid (Like a blacklisted name), lastly stores this new information on **RD** Database, along with a new Human Warrior [**Game state**] **(GS)** on the **Realm Game State (RGS)** database.

The **Game state** or **GS** will be explained in detail in the next chapter, but it may be understood as the current data for a player in a determined time, like position, equipment, health, mana and more, but the **GS** also represents all the &quot; **sensory**&quot; data that the player receives, like players and creatures around his **Field of View** (or **Region of interest** ( **ROI** )), animations, sound effects, spells and every action happening around it.

Then **RWAM** loads a new **GS** for a player based on his new character and because Matt chose a Human warrior, **RWAM** places his new character in a specific place in Elwynn Forest (Starting region for humans in WoW) along with warrior starting gear and warrior character traits.

Then after finishing storing the data in the **RGS** , the **RWAM** responds with successful character creation, along with character information to show the **Character choosing screen,** Matt now chooses his warrior and clicks on &quot; **Enter World&quot;**.

1.
## Now It just got Complex 😱

Until now we discussed on a high level the steps to enter the game world, the mentioned steps are important but simple to understand, the next steps may involve some game logic to explain the inner complexity of making a freaking anvil appear to a player, without crashing everything.

- Entering the World
- Game state (the data to refer to a single moment in a game)
- Loading dynamic objects (NPCs, Players, Mail Boxes, Anvils...)
- Region of interest and brute forcing
- Important information and client side information

Add in specific later

- Queues and auto-scaling DDOS
- Secure channel nounce or sequence packet number

Refereces and more reading

- [https://medium.com/@zendar/the-anatomy-of-an-mmo-behind-the-curtains-of-darkfall-47e1391d7710](https://medium.com/@zendar/the-anatomy-of-an-mmo-behind-the-curtains-of-darkfall-47e1391d7710)
- [https://developer.valvesoftware.com/wiki/Source\_Multiplayer\_Networking](https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking)
- [https://www.gabrielgambetta.com/client-server-game-architecture.html](https://www.gabrielgambetta.com/client-server-game-architecture.html)
- [http://fileadmin.cs.lth.se/graphics/theses/projects/mmogarch/som.pdf](http://fileadmin.cs.lth.se/graphics/theses/projects/mmogarch/som.pdf)
- [https://cloud.google.com/solutions/gaming/cloud-game-infrastructure](https://cloud.google.com/solutions/gaming/cloud-game-infrastructure)
- [https://serverfault.com/questions/12590/what-server-architecture-is-appropriate-for-a-multiplayer-online-game](https://serverfault.com/questions/12590/what-server-architecture-is-appropriate-for-a-multiplayer-online-game)
- [http://ithare.com/nosql-vs-sql-for-mogs/](http://ithare.com/nosql-vs-sql-for-mogs/)
- [https://www.reddit.com/r/gamedev/comments/4q5587/multiplayer\_online\_game\_architecture/](https://www.reddit.com/r/gamedev/comments/4q5587/multiplayer_online_game_architecture/)
- [https://www.ibm.com/developerworks/library/ar-powerup1/](https://www.ibm.com/developerworks/library/ar-powerup1/)
- [https://www.cs.ru.nl/bachelors-theses/2006/Martijn\_Moraal\_\_\_0131903\_\_\_Massive\_Multiplayer\_Online\_Game\_Architectures.pdf](https://www.cs.ru.nl/bachelors-theses/2006/Martijn_Moraal ___0131903___ Massive_Multiplayer_Online_Game_Architectures.pdf)
- [https://patents.google.com/patent/US6152824A/en](https://patents.google.com/patent/US6152824A/en)
- [https://gamedev.stackexchange.com/questions/129674/multiplayer-game-servers-architecture](https://gamedev.stackexchange.com/questions/129674/multiplayer-game-servers-architecture)
- [https://core.ac.uk/download/pdf/38103454.pdf](https://core.ac.uk/download/pdf/38103454.pdf)
- [https://ieeexplore.ieee.org/document/1417631](https://ieeexplore.ieee.org/document/1417631)
- [https://www.researchgate.net/figure/Different-gaming-architectures-a-In-a-client-server-architecture-the-server-is\_fig2\_262233529](https://www.researchgate.net/figure/Different-gaming-architectures-a-In-a-client-server-architecture-the-server-is_fig2_262233529)
- [https://www.researchgate.net/publication/220831981\_Colyseus\_A\_Distributed\_Architecture\_for\_Online\_Multiplayer\_Games](https://www.researchgate.net/publication/220831981_Colyseus_A_Distributed_Architecture_for_Online_Multiplayer_Games)
- [https://dspace.library.uu.nl/handle/1874/259124](https://dspace.library.uu.nl/handle/1874/259124)
- [https://www.sfu.ca/~rws1/papers/Cloud-Gaming-Architecture-and-Performance.pdf](https://www.sfu.ca/~rws1/papers/Cloud-Gaming-Architecture-and-Performance.pdf)
- [https://stackoverflow.com/questions/6641364/online-browser-game-architecture](https://stackoverflow.com/questions/6641364/online-browser-game-architecture)
- [https://stackoverflow.com/questions/42840589/server-architecture-for-simple-real-time-online-game](https://stackoverflow.com/questions/42840589/server-architecture-for-simple-real-time-online-game)
- [http://www2.ic.uff.br/~celio/classes/mmnets/slides/mmog05.pdf](http://www2.ic.uff.br/~celio/classes/mmnets/slides/mmog05.pdf)
- [http://highscalability.com/eve-online-architecture](http://highscalability.com/eve-online-architecture)
- [https://www.gamasutra.com/blogs/MorisPreston/20181126/331312/Scalability\_How\_to\_scale\_your\_app\_or\_online\_game\_in\_terms\_of\_architecture\_and\_hosting\_infrastructure.php](https://www.gamasutra.com/blogs/MorisPreston/20181126/331312/Scalability_How_to_scale_your_app_or_online_game_in_terms_of_architecture_and_hosting_infrastructure.php)
- [https://pdos.csail.mit.edu/archive/6.824-2005/reports/assiotis.pdf](https://pdos.csail.mit.edu/archive/6.824-2005/reports/assiotis.pdf)
- [https://www.quora.com/How-do-I-design-a-server-architecture-for-an-online-game](https://www.quora.com/How-do-I-design-a-server-architecture-for-an-online-game)
