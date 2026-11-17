Before DNS, during the days of the ARPANET there was a single monolithic file developed by SRI International who also distributed files from a single host point called SRI-NIC. This was the host table, also called host.txt
![[Pasted image 20260527182848.png]]
Each line contains information about a single host and its IP in second field,. its canonical or official name in the beginning of the third field and optionally followed by one or more aliases, then the hardware that ran the host, the operating system it ran and finally a list of services it supported 
You may notice that the IPs are not ordered in anyway. This is because network admins on the ARPAnet would submit their portion of host.txt to the NIC which would aggregate them into a single file and make it available thru ftp. 
As the arpanet grew, this became an issue\
- consistency/latency
	- the host.txt file was only updated twice a week so sometimes it would be days before your new host was accessible over the ARPAnet
- Traffic and load on the SRI-NIC network
	- The bandwidth was significantly consumed by people uploading and downloading their host.txt files
		- Back then the pipes that constituted ARPAnet ran at 56kbps 
	- IP addresses could be doled out without contacting a central authority but host names couldn't
		- In order to prevent collision, the Network Admin had to manually check the host.txt file for name collisions 
	- Searching the file was linear O(n)
		- Some company host.txt files were hundreds of thousands of lines long
		- The ARPAnet host.txt file was far far longer
		- Searching was done line by line since nothing was **sorted**
			- Some searches timed out before even getting halfway
		- Some administrators began to sort the file so the most commonly looked up hosts were on top

The chief design goals were
- To allow local administration of partitions of an overall host-naming database while ensuring global availability across the ARPAnet
- To use a hierarchical namespace which would allow easy partitioning of the database and ensure the uniqueness of hostnames
![[Pasted image 20260529080233.png]]

## DNS
The internet's mapping service. 
	Eg: For email addresses, the DNS only cares about the address after the @ sign. `info@example.com` -> example.com
	Eg: For urls, only the portion after https:// and before the second / is relevant. `https://www.example.com/` -> www.example.com
Humans find it far easier to work with names than IP addresses.
DNS is also a distributed database which is partitioned into **zones**. These zones can be managed in a distributed and decentralized way while being available globally across the internet.
	Organizations connected to the internet can manage their own zones almost autonomously while staying available globally
Databases often have indexes and the DNS is no different.
- The DNS's indexes are those *funny dotted* names which are formally called domain names
- Domain names are actually paths in an inverted tree
#### Inverted Trees
An inverted tree is a data structure made up of nodes and links between nodes.
Each node is connected to its parent by a single link and one node can be parent to arbitrarily many children
![[Pasted image 20260529084556.png]]
Each node has a label between 0-63 bytes in length. 
The root node situated at the top has a special reserved label written as "" (for a zero-length label) -> You have to imagine the label as being hidden between those double quotes
The main restriction is that all of the children of a node have to have different labels

One label isn't enough to uniquely identify a node in the tree. In order to do that we need a **Domain Name**, which is a series of labels from the node to the root (read with dots separating each label)
![[Pasted image 20260602152736.png]]


### Domains, Delegation and Zones
- Domains are groups of related nodes. They're a subtree of the namespace
- Domains are identified by a domain name, the domain name of the node at the top of the subtree
![[Pasted image 20260602153234.png]]-
	- Here the node identified has a *com* domain name. The red triangle represents all the domains in the *com* namespace ,aka, domains that end in *com* (like google.com, hp.com, yahoo.com,etc)
	- This includes all of the nodes **at** or **below** *com*'s namespace
#### Subdomains
Subdomains are domains that are contains within another domain. 
The apex or top of the sub-domain still lies within the boundaries of the parent domain. 
	- The name of the sub-domain will always end in the name of the parent domain
![[Pasted image 20260602154328.png]]
#### Domain Management
Domains are managed by a single organization.
	- ICANN controls the **root domain**
	- Verisign controls the **com domain**
		- the com domain is a sub-domain of the main root domain so how can two companies control the nodes at and below *com*?
		- this is where **delegation** comes in
#### Delegation
Delegation allows an organization to assign control of a subdomain to another organization
Instead of now containing all of the information of the sub-domain, the parent domain only contains **pointers** to sources of data in the sub-domain
The parent and child domains are now separate administrative containers called **zones**
![[Pasted image 20260602155048.png]]
	- The red lines represent delegation
		- The orange triangle is the root domain managed by ICAAN
		- The red triangle is the com domain managed by Verisign 
			- Below the com domain are there hundreds of millions of further delegations 

#### Internet Namespace History
Initially the internet namespace was simple
	- There were 7 top level domains
		- com -> For commercial orgs
		- edu -> for educational orgs
		- gov -> for US gov orgs
		- int -> for international treaty orgs
		- mil -> for US military orgs
		- net -> for networking orgs
		- org -> for non-commercial orgs
These restrictions were relaxed on com, org and net domain names. Any organization may now procure their use. These were the first of the Generic Top Level Domains (GTLDs)
Besides these, we also had a "transitional" domain called *arpa*
	- Hosts moving from the host table into DNS were initially **hostname.arpa**. These domains later served as an alias for the host's new domain name. Eg: sri-nic became sri-nic.arpa and later nic-ddn.mil. However, the sri-nic.arpa domain name stayed as an alias for the dns domain for quite a while
	- While it was intended to be transitional, the arpa domain name stayed along for quite a while and is still used today albeit for diff purposes
`The reason why the US has so many privileges in DNS naming schemes is because DNS itself was created to replace the ARPAnet, which was a US Department of Defense project`
Later, top level domains were reserved for every country in the world called **country code Top Level Domains** or **ccTLD**s
	- These labels were taken from a document called ISO 3166
		- Most of these are in use.
		- Eg: Nepal(np), China(cn),Germany(de),India(in),etc

#### Internationalized Domain Names
All the DNS domains so far have used simple ASCII characters in their labels. 
However, a great majority of the world speak languages that don't use ASCII characters. Thus, this leads to some unique TLDs
![[Pasted image 20260602165003.png]]
We cannot cater to these languages directly because DNS software assumes each bite in a label represents a single character as an ASCII one. However, it would take tens of thousands of bytes to represent the characters of the world. 
	- Instead we encode these special characters from non-ASCII scripts into ASCII using a special algorithm called Punycode
	- This works sort of like encoding binary data into ASCII using base 64
![[Pasted image 20260602165826.png]]
	- Notice how these encoded domains begin with **xn- -**
	- No other label should start that way
##### Internationalized Domain Names in Applications
Web Browsers and other software, that support a standard called Internationalized Domain Names in Applications (IDNA), take characters in other scripts in URLs and encode them using Punycode
	- This allows us to enter URL in these non-ASCII characters and still get to the right place
	- You can enter sub-domains with these non-ascii characters and the browser will automatically convert them into punycode
### Name Servers, Zones and Authority
DNS is a *client-server* system. The servers are called **name servers** or **DNS servers**. The clients are called **resolvers**, they query name servers. 
	This is the heart of DNS
#### Name Server Load Zones
Name servers load data in Zones (administrative units using delegation)
	- A Name server may load thousands of zones but none load the entire namespace
	- A Name Server that loads an entire zone is the **authoritative** server for that zone
	- Some name servers don't load any zone (discussed later)
![[Pasted image 20260603165531.png]]
Name servers are responsible for answering queries from the resolvers as well as other name servers
	DNS isn't just a simple client server system where the only communication happens between the client and server. Sometimes, servers talk to other servers as well.
#### Resolvers
Resolvers are the client half of the domain name system. However, unlike the server side, they aren't usually a separate piece of software
	- Usually they're built into the operating system like Windows or Mac.
	- Even simpler devices that support DNs lookups have resolvers built in their firmware 
What is the resolver for? -> OS exist to run apps.
	- Those apps need to look data stored in DNS. The resolver gives them a simple way to do that
	- Resolvers take application requests for data in DNS and translate them into DNS queries
		- The resolvers awaits a response and it may have to retransmit the query if it times out (no response within a timeframe)
		- The resolver may also query different name server if the first one is not responsive.
	- When resolvers receive a response, the resolver will unmarshall that into a data structure and present that data structure back to the application
	- This is necessary because the low-level format of a DNS message is pretty bad -> Variable length and binary 
		- Without this, just putting together a syntactically correct query would require mastery of the DNS protocols which is pretty time consuming
		- Resolvers hence obviate the need for applications to understand it
			- Resolvers in general are simply libraries of software routine that abstract the low-level details of assembling a query, sending it to a name server, resending it if required and extracting the information of interest from a response
#### Resolution
Name servers host a lot of zones. Thousands even. However, it is but a sliver of the name space. 
Resolvers could ask any name server about any arbitrary domain names without regard to it's governing zone. How would this work?
	- The entire name space is designed as an inverted tree and *resolvers specify the domain name they're interested in*, in their queries
	- This allows resolvers to start at the top and make it's way down to the name servers it's after
		- There is a set of name servers authoritative to the root zone called the **Root Name Servers**
		- These servers can be queried for any domain name. However, unless we query it for information specifically in the root zone, they will not know the answer. They can however, point us to the correct authoritative name servers further down the chain.
			- `Query,Refer,Repeat`
		- Eventually, the querier reaches the name server authoritative for the zone that contains the domain name it's interested in
		- That name server answers the query
		- The first name server, the one that did all the work then relays the information to the resolver
This process, whereby one name server queries a series of other name servers to find arbitrary data in the namespace is called **Resolution**

![[Pasted image 20260607152517.png]]
How this basically plays out is that our devices also called **stub resolvers**(called resolvers in our diagram) issues a recursive query to the actual **Recursive Resolver** that is maintained by our ISP(called name server in this diagram). Our devices are not actual resolvers. Its only function is to delegate. If the recursive resolver does not have the answer in it's cache, it queries the root domain which points us to the authoritative name server of our query. The recursive resolver does most of the heavy lifting 
![[Pasted image 20260607155428.png]]
We may notice that the first name server(recursive resolver) did most of the work. Why did the rest of the servers get off so easy? 
	This depends on the type of query they were handling
	The name server accepted a **recursive query** which obliges it to keep doing the required until an answer is found. 
		Name servers may nor may not send referrals
	A name server responds to a **non-recursive query** with the answer it best knows, usually a referral. 
	Most resolvers send recursive queries because they are not smart enough to do it themselves
	Most name servers are smart enough to follow referrals so they **send** non-recursive queries by default. That gives the name servers that receive them the option to reply with a referral. 
		It is in fact beneficial to name servers to send non-recursive queries as these referrals contain valuable information about the namespace.
#### Queries and Responses
Queries and Responses are types of DNS messages based on UDP(User Datagram Protocol)
Both of them also share a common format
![[Pasted image 20260607172328.png]]
UDP is known to be an unreliable but fast protocol. 
	Since both DNS Queries and Responses are based on UDP, resolvers and name servers must take packet loss into account and retransmit when needed.
Queries and Responses contain and Header section, a Question section and a Response section.
	Response sections may contain an answer section, an authority section and an additional data section
		- The Answer section gives you the exact information you explicitly asked for (`google.com` is located at `142.250.190.46`)
		- The Authority section tell you exactly who officially owns and manages the records for that domain (This domain is officially managed by the authoritative name server `ns1.google.com`)
		- The Additional section contains extra, helpful information that you didn't explicitly ask for but the server gives it to you anyway (By the way, if you ever need to talk directly to that manager server (`ns1.google.com`), its IP address is **`216.239.32.10`**.)
			- This is also called `Glue Record`.
		- Answer, Authority and Additional sections are just there to carry data in the form of **Resource Record**s (RRs)
			- That is what we store in Namespace.
			- All those domain names are just indexes pointing to records
The Header and Question section are always present but the response section varies
	**- Standard Query (All three are omitted)**
		- When our device first sends a query. All fields blank because we are asking a question
	**- Name Error Response (Answer and Additional omitted)**
		- Typing in a website name that doesn't exist, the DNS server sends a functional error response known as an NXDOMAIN error
	**- Standard Cache Response (Authority and Additional Omitted)**
		- When a recursive resolver already has an address in its cache
	**-  Referral Response (Answer omitted)**
		- When a recursive resolver asks a root server or a TLD for a domain but they don't have the final IP Address
	**- No Data Response (Answer and Additional Omitted)**
		- Suppose a website exists perfectly but only has an IPv4 address (`A record`) and we query for an IPv6 address (`AAA record`), the server will return a successful status code but will leave the payload empty
The Header section contain and OPCODE which for queries is just QUERY
```
OPCODE = QUERY
```
We also have a QR(Query/Response) bit which is 0 for a query and 1 for a response
We also have a RD(Recursion Desired) bit which is 0 for non-recursive queries and 1 for recursive queries

The Question section contains a 
	- Domain name which contains the domain name the querier is interested in. 
	- The Class which is almost always **IN** for **I**nter**N**et
		- DNS was designed with the idea that it could accommodate different kinds of networks, not just TCP/IP based internets.
		- Those other kinds of internets would've used different classes
			- But other types never really took off
		- So except for a special number of cases, Class is always **IN**
	- Type is the type of data (record) the querier is interested in
```
Domain name: Interested Domain Name
Class = IN 
Type: Interested data type (record) [IPV4 addresses, IPv6 addresses, mail routing info,etc]
```

#### Caching
Once you've looked up the address of a website, the recursive resolver will cache the resource records of that website. 
	- For example, we requested the address of www.infoblox.com in the above image. The recursive resolver requested information about it from the root server. The root server pointed it to the com name server and that pointed to the final destination.
		- Here the recursive resolver will cache the names and addresses of the com name servers, the names and addresses of the infoblox.com name servers. 
		- It will also cache the address and answer of infoblox.com
All websites set a TTL(Time To Live) for their DNS records. After the count hits zero, the record is removed from the Recursive Resolver's cache. However, the recursive resolver itself has a limited cache size, if it fills up, the recursive resolver will begin removing records based on last time it was accessed. Unpopular and infrequent queries will be the first to go. If one of such resource record is requested, it must query the root server again.
	- This means that if we query support.infoblox.com, the recursive resolver already has the names and address of the infoblox.com name server which allows it to send back an answer directly without further query into the namespace
		- This cache also helps other sites because we already have the names and address of the com name server, so the recursive resolver can skip querying the root name server and directly ask the com name servers

#### Time To Live (TTL)
It is not appropriate to cache resource records forever because sooner or later, somebody might need to change the address of a web server or add/decommission a name server. Thus, each recourse record has a Time-To-Live. TTLS tells a name server how long it can cache the specific resource record. The name servers decrement resource record TTLs until it is zero after which it is deleted
	- TTLs range from seconds to hours to days
	- The administrator of the zone that contains the record decides the TTL
	- TTL ,while present in an authoritative name server for a zone, has no effects on it. 
		- The authoritative name server will assign a TTL to a record when requested but the TTL count down will only begin decrementing when it leaves the authoritative infrastructure

#### Server Selection and Retransmission
All zones have more than one name server. Thus, it is upto the name server handling the recursive query to choose which root name server to query, which com name server to query and so on. 
Since DNS Queries and Responses are transferred over UDP, it is possible that they don't arrive. Both resolvers and name servers must handle retransmissions
##### Server Selection
The various makes and models of name servers choose which name server to query based on different algorithms.
The Open Source BIND name servers(one of the most popular implementations) uses roundtrip tp choose
	- They time how long a response takes and remembers that
	- They favour the quickest name server to respond
		- Whenever these servers have a choice of authoritative name servers to query, they always choose the fastest
	- If it has never queried a name server before, it just give it a random low roundtrip time
		- Low so it'll atleast try to query it before playing favourites 
		- Random so it doesn't try all of the authoritative name servers for a zone in the same order as every other BIND name server
	- This algorithm which has been present since the 80s is remarkably effective
##### Retransmission
Like server selection, retransmission algorithms are also dependent on name server implementations. However, all name servers send queries, await response and retransmit the query(to another name server if possible) if they don't hear back (receive a timeout)

#### Authoritative, Recursive and Caching Name Servers
An Authoritative Name Server is a name server that is authoritative for one or more zones. However, this doesn't narrow down much as almost all name servers are authoritative for some zones. 
What most people mean by **Authoritative Servers** are name servers whose main function is to answer non-recursive queries for domain names and zones that it is authoritative for.
	- Some might even have recursion disabled
		- This means that even if they receive a recursive query, they treat it as a non recursive query
Another situation in which Authoritative Name Server is meaningful is when you specify the zone or zones that its authoritative for. 
	- When you specify the authoritative names for a website say google.com, we know exactly which name servers you mean

A **Recursive Name Server** on the other hand is a name server that is willing to handle recursive queries. It will do all the necessary to find the answer
Virtually all Recursive Name Servers also cache their responses, both referrals and answers that they receive from other name servers so they're also called **Caching Name Servers** too. Those two names are **interchangeable**. 
Some recursive name servers are basically dedicated to the task of performing recursion. They are not authoritative to any zones or only a handful of built-in zones(like the one that maps the loopback address to local host)
	- While these servers do not host public websites, they still maintain a few mandatory internal structure files
		- Every computer uses 127.0.0.1  to refer to itself as a localhost
		- To prevent your computer from accidentally reaching out to the open internet just to find itself, the recursive server has a small hardcoded built-in-zone that instantly translates `lovalhost` to `127.0.0.1`
	- These are often called caching-only name servers
Some recursive name servers may be authoritative to many many zones

Authoritative Name Servers come in two sub-divisions, Primary and Secondary
The difference is where they get their zone data.
![[Pasted image 20260609222249.png]]
A Primary Name Server reads zone data directly from a file which is called a zone data file.
A Secondary Name Server transfers its zone data from another authoritative name server, often a zone's primary name server but sometimes from another secondary as well. 
	- This mechanism is called a Zone Transfer
	- Like queries and responses, Zone Transfers are carried in DNS messages
	- A Zone Transfer is really just a special case of a query that can generate a large response
		- This is carried out in a series of DNS messages over TCP instead of UDP
The secondary name server initiates the zone transfer by sending its *master server* (the authoritative name server that it is configured to pull zones from), a special type of query called an AXFR query. 
	- The master server responds by sending the resource records in the zone
After the secondary server has a copy of the zone, it checks on its master server periodically at what's called a refresh interval to determine whether the zone (serial number) is changed.
Each zone contains a single serial number that is changed any time its data changes
	- A zone's serial number is an indication of the version of the data in that zone
If the serial number on the master is higher than the serial number on the secondary, the secondary requests a zone transfer
	- The secondary name server requests an SOA(Start of Authority)
	- The primary responds with the serial number
	- The secondary compares the serial number of the primary's response with the one that it holds in its secondary copy of the zone.
	- If they're upto date, the secondary wait for the refresh interval and then checks again
![[Pasted image 20260613140431.png]]

 There is no difference in authority between a primary and a secondary. Just as a primary marks the response that it sends from its own data as authoritative, so does a secondary.
	 In fact, if you look at a set of authoritative name servers for a zone on the internet, you can't always tell which one the primary is. 
In practice, DNS provides primaries and secondaries to make it easier to run multiple authoritative name servers for a zone from one place. 
For robustness and performance, zones need atleast two and possibly many more authoritative name servers. 
Its much easier to use the built-in zone transfer mechanism to replicate zone data instead of coming up with your own means.
Calling a name server primary or secondary without specifying the zone(s) is a little imprecise.
A single name server can be primary for some zones and secondary for others.
However, in common usage a primary name sever is one that is primary for most of the zones it is authoritative for and a secondary is a name sever that is secondary to most of its authoritative zones
What primary and secondary DO NOT mean is the first or second name servers our recursive resolvers queries. It is not ordering. This is a common misuse. 
Primary Name Servers
	- Also called Read-Write Master
	- Original Source of Truth for a domain's DNS zone record
	- Only server where a network admin can manually add, delete or change website records
	- Defines the baseline SOA record
		- Every time an admin makes changes, the primary server increments the serial number inside the SOA record to update that an update has occurred
Secondary Name Servers
	- Read Only Replica
	- Strictly read-only back-up copy hosted on an entirely separate physical server
	- Cannot be logged into bt admins to make changes
	- automatically updates information by checking the primary server's SOA serial number by relying on the automation timers built into the SOA record itself
		- If a primary server's SOA number is greater than its(secondary server) own, it automatically pulls a fresh mirror copy of the database. 
			- This is called a **Zone transfer**
	- If for some reason, usually due to human error, the serial number of the primary server is less than the serial number of the secondary server then it simple just stops working.
		- The internet still works but if a recursive resolver were to query a secondary server as opposed to a primary server, then it will only show it's cached records (updates no longer take place)
			- You end up with different variations of the same website 

#### Forwarding
Forwarding is an alternate method of resolution. In this method, the recursive name server is lazy.
If the recursive resolver itself doesn't know the answer to a query(from authoritative zone data or cache), instead of going through referrals and finding the answer itself, it simply forwards the recursive query to another name server called its forwarder
If the forwarder accepts the recursive query, it is upto it to follow through the referrals and find the answer
![[Pasted image 20260613153836.png]]
	Note: A forwarder is a name server that **receives** a recursive query from the lazy name server. The name server that actually forwards the query could be called a *forwarding name server* but there is no official term for it.

What is forwarding used for?
	-  In the old days of DNS, it gave admins a way to funnel their queries through a single super-name server which would build up a rich cache of data
	- In the modern day, its most commonly used in a company's firewall
		- Rather than poking holes in the firewall to allow all internal name servers to query name servers on the internet directly, most internal name servers are configured to forward their queries to designated forwarders.
			- These forwarders are allowed to send queries to the internet and relay it back
		- Using forwarding, all internal name servers are able to resolve internet domain names but not all name servers are exposed to the internet

#### Makes and Models: BIND, Windows DNS Server & More
Bind was released in 1984 and is more venerable than the Windows DNS Server which came out in 1996
##### BIND
- Stands for **Berkeley Internet Name Domain**
- Open source
- Originally developed in 1984 as part of BSD Unix (Berkeley Software Distribution)
- The first version released was version 4 (They were trying to sync up release numbers with other software being released in BSD Unix)
- Maintenance of BIND was relinquished to ISC (Internet Software Consortium) in the 1990s
	- ISC released a rewritten version of BIND called BIND 8 (version 8) in 1997 and another rewrite called BIND 9 in 2000.

**Windows DNS Server**
- Shipped with all Windows Server operating systems
- Used to be called Microsoft DNS server

**Differences between BIND and Windows DNS Servers**
BIND
	- BIND is highly configurable and feature-rich
		- Configuration is largely done through *named.conf* and zone data
	- Extremely popular as it powers a "double digit" percentage of the name servers in the namespace
Windows DNS Server
	- Very Popular as an internal name server
	- Ships with a GUI for managing DNS called the DNS Manager
	- Integrates with Microsoft's Active Directory
		- It can store its files in active directory and read its config from active directory

**Other Name Servers**
	- Both BIND and Windows DNS Servers can function as recursive, authoritative or both at the same time
	- Recently some specialized servers have been developed that only support recursion or authority 
	- Two of the best come from NLnet Labs:
		- Their recursive name server is called Unbound 
		- Their authoritative name server is called NSD
![[Pasted image 20260615202050.png]]

#### Resource Records
These are units of data stored in the namespace.
They map domain names to IP addresses, mail routing information and more.
Referrals consists of resource records. But what do they look like?
	Inside a DNS message, Resource Records appear in a **binary format**
	In zone data files and in the output of better DNS query tools, they have a textual format called the **presentation format** or the **master file format**

##### Presentation Format
Resource records in presentation format begin with
![[Pasted image 20260616172332.png]]
	Domain Name -> Domain in the namespace to which the record is attached
		- If that domain name ends in a dot, its absolute 
			- Think of a path name in a file system that begins with a backslash
		- If it doesn't have a dot, it maybe relative and may have another domain name called the **origin** appended to it
			- By default, the origin in a zone data file is just the domain name of the zone
			- When a network admin configures an authoritative primary zone file, they define the origin at the very top
```
$ORIGIN infoblox.com
```
~
			- Once this origin name is established, it acts as a shortcut for the rest of the config records. If a record is written without a trailing dot (which makes said record relative), the DNS software automatically stitches the origin name onto it.
			- E.g: if you add a record simply named `www`, the system automatically interprets it as `www.infoblox.com`
			- Once you've set up your origin name, you can use `@` as a shortcut or macro to point to it(origin name)
```
E.G:

; --- 1. The Mandatory Start of Authority Record ---
@       IN      SOA     ns1.infoblox.com. admin.infoblox.com. (
                        2026061601   ; Serial Number (YYYYMMDDNN)
                        86400        ; Refresh
                        7200         ; Retry
                        3600000      ; Expire
                        172800 )     ; Minimum TTL

; --- 2. Name Servers for the Domain ---
@       IN      NS      ns1.infoblox.com.
@       IN      NS      ns2.infoblox.com.

; --- 3. Mail Exchange (MX) Record for Corporate Email ---
@       IN      MX  10  mail.infoblox.com.

; --- 4. Core Root Domain Mapping ---
@       IN      A       192.0.2.1

; --- 5. Relative Subdomain Records (No trailing dots) ---
www     IN      A       192.0.2.2
ns1     IN      A       192.0.2.3
ns2     IN      A       192.0.2.4
mail    IN      A       192.0.2.5
```
~
	TTL -> Time to Live, *optional field* 
		- How long recursive name servers can cache this record
		- TTL is natively an integer number of seconds but most systems will let you use scaled values like 1h, 10m, 1h30m, etc.
			- Values are usually minutes or hours but for some records it could be days
		- If an explicit TTL is not included, the records receives a default TTL
	class -> *optional field*
		- almost always **in** for **internet** so its the default
	type -> type of record
		- A mnemonic indicating what type of data is stored in that record
		- E.g: `A` -> IPv4 Address, `AAAA` -> IPv6 Address, `SOA` -> Start of Authority, `NS` -> Name Server, `MX` -> Mail Exchange
	RDATA -> Record specific DATA
		- the format of the RDATA depends on the class and type
		- E.g: an A record which stores IPv4 addresses in the IN class requires a dotted octet representation of an IPv4 address as its RDATA
![[Pasted image 20260616171252.png]]
		- E.g:  an MX record for mail routing requires a 16-bit preference value(priority indicator) followed by the domain name of the mail exchanger
![[Pasted image 20260616171451.png]]
***Note**: mail.infobox.com. is a Fully Qualified Domain Name. This means that this is an absolute path within the global Domain Name System hierarchy*

##### Resource Record Sets (RRset)
A resource set is simply a set of resource records that share the same domain name(owner), class and type
These types of record names deserve a separate name because they are inseparable 
They are generally grouped together in zone data files for administrative convenience, but they're always returned together since there is no way to specigy for example if you only want the first or second A record for www.infoblox.com

#### The A Record
The most common and arguable the most common record type
The A originally stood for address 
They are IPv4 address records
![[Pasted image 20260616181244.png]]
It takes as its record specific data a single IPv4 address in dotted octet notation 
You can map a single domain name with multiple IPv4 Addresses by simply attaching multiple A records to the same domain name (which we may need for routers or multihomed hosts)
	- But how do we know which address an application uses first when it looks up a domain name?
		- It is generally the first one returned 
		- Only some applications try to choose from a list of addresses
		- Most will try to cycle through them if there are more than one
	- **It is not necessary for the first mentioned address in a zone data file to be the first one to be returned.** The order can be configured and changed.

#### The AAAA Record
Called **Quad A Record** not AAAA 
It has four As because the 128-bit long IPv6 address is 4 times as long as the 32-bit IPv4 address. 
The only difference between an A record and an AAAA record is the record specific data
	Quad A Records require a single IPv6 address in the R data (that colon separated address)
If you want to attach multiple IPv6 addresses to a domain name, you would use multiple Quad A records (just like for A Records). 
![[Pasted image 20260617145601.png]]

If you attach both the A Record and the Quad A Record to the same domain, which one will get used first?
	This depends on the client. Some clients will always use one record type before the other, typically Quad A before A.
	Some clients will lookup both addresses simultaneously and will try to connect both IPv4 and IPv6 (if they're available)
		They essentially race both addresses together to see which one responds the fastest and use that one
		This algorithm is called Happy Eyeballs
#### The PTR (PoinTer Record)
It is chiefly used for mapping IP Addresses (IPv4 or IPv6) back to a domain name.
	This is called **reverse mapping**
The A and Quad A Records are not very useful for reverse mapping
	Its easy to find an IP Address given a domain name but its difficult to find a domain name from an IP Address. Exhaustively searching the entire namespace is not feasible
	There is a special part of the name space in which the octets of an IPv4 address or hexadecimals of an IPv6 address are used as the labels of the domain name 
		This makes it easy yo determine which domain name to lookup when you're given a IP Address
		The type of record you look up for that domain name is a pointer record
	The special part of the namespace is called the ARPA TLD (originally supposed to be transition from the host-table to DNS)
		ARPA has two subdomains for mapping
			in-addr.arpa -> IPv4 reverse mapping
				The labels of a domain name are the octets of an IPv4 address in reverse order
				E.g: If the IP Address of a website is 192.1.2.10, the reverse pointer record is stored as 
				10.2.0.192.in-addr.arpa.   IN   PTR   mail.example.com.
			ip6.arpa -> IPv6 reverse mapping
				The labels of a domain name are the hexadecimals of an IPv6 address in reverse order
				E.g: If the IP Address of a website is 2406:da00:ff00::3213:5554, the reverse pointer record is stored as
				4.5.5.5.3.1.2.3.0.0.0.0.0.0.0.0.0.0.0.0.0.0.f.f.0.0.a.d.6.0.4.2.ip6.arpa
		The octets and hexadecimals are reversed to make delegation work sensibly. DNS records read from right to left but IP Addresses read from left to right.
			This is because IP addresses place the most general information in the right and the most specific information in the left
			E.g: Think of how security . labs . example . com and 192 . 168 . 10 . 55 are handled
![[Pasted image 20260619210240.png]]
What is reverse mapping used for? -> Some software reverse map IP Addresses to log domain names of clients that connect. Some check to see if the clients are authorized(that is a bad idea because the fundamental flaw of DNS is that whoever owns the IP address controls its PTR Record. Assuming you configure your firewall or application to grant access to anyone whose IP resolves to www.example.com, an attack can simply acquire any public ip address and their their own PTR Record to say www.example.com and be granted access. PTR records prove what an IP claims to be rather than what they actually are)
You can attach multiple PTR to a single domain name but like A and Quad A Records, there is no guarantee on the order in which they will be returned
Many clients that use pointer records do not expect more than one and thus do not search through them so this may not work reliably 

#### CName Record (Canonical Name)
CName Records create an alias in DNS.
	The alias is the domain name to which the domain name is attached
	The target of the alias (called the official or canonical name, CName) is the record specific data
![[Pasted image 20260620104507.png]]
CName Records are *singleton* records
	If a domain name has a CName Record attached, it may not have any other record types
		That makes sense because by saying that a given domain name is an alias for another, the CName Record is also saying that all those other records should be attached to the canonical name, not the alias
CName Record are processed differently compared to the rest of the records	because a querier doesn't know ahead of time that a domain name is an alias.
	For example: A Querier may ask for an A Record for the alias and an explicit search for an A Record can't find one because its attached to the canonical name
	The authoritative server that receives a query for any type of record for a domain name and doesn't find one will then look for a CName Record. 
		If the CName Record exists, regardless of the record type sought in the query, the authoritative name server will return it
	When a recursive query sees the CName Record, it will restart the query replacing the alias with the canonical name and the records are resolved as required 
		It is a bad idea to chain aliases because it increases the work load involved in resolution

#### MX Record
MX Record or Mail Exchanger Record is used by mail servers, specifically SMTP Mail Transport Agents(MTA)
The format is fairly simple
![[Pasted image 20260620113414.png]]
	The owner name is a domain name that appears to the right of the @ sign in an email address.
	The record specific data consists of 2 fields
		Preference value -> Unsigned 16-bit integer written in decimal (0-65535)
		Mail Exchanger -> Domain name of the mail exchanger
A mail exchanger is a mail server that accepts mail addressed to the domain name that owns the MX Record. 
	Accept means that it could be delivered to the recipient or it could simply replay the mail to another main server 
The preference is an indication of the desirability of delivering mail to the corresponding mail exchanger.
	**Lower** preference values indicate more preferred mail exchangers. (0 is the most preferred)

##### The MX Delivery Algorithm
This is how MTA determines how to deliver mail address to a user at a non-local domain name 

- The MTA looks up the MX Records for that domain name 
- It then sorts the MX Records based on their preference (0 being the highest and 65535 being the lowest)
- The MTA looks for the domain name of the host it's running on in the list 
- If it finds the host in the list it removes it. It also removes any record with a preference value equal or lower than its domain's value. ->Helps avoid loops because sending mail to itself will not help mail get delivered and sending a mail to a less preferred mail exchanger would just be sending it away farther away from its destination 
- The MTA should now have a list of records that it can manage. If no records remain, it is an error
- The MTA attempts to deliver the mail to the most preferred remaining mail exchanger
	- If there are more than one preferred mail server, it can choose among them
- If the delivery fails, the MTA must try other mail servers of that same preference before trying any less preferred mail servers 
- If a mail exchanger tries looking up MX records for a destination and doesn't find any, it'll usually try looking up A and Quad A Records for the destination too and treating any it finds as equal preference mail exchangers
	- If the MTAs discover valid address records, eg: `example.com. IN A 192.0.2.25`, it treats that target IP address as an implicit mail exchanger
	- The rule of equal preference comes from the MTA falling back to using the root domain's address records which bypasses the traditional 16-bit preference field entirely. Thus all IP Addresses are implicitly considered of equal preference

#### SRV Record (Service Record)
MX Records are only useful for delivery of email using SMTP. However some aspects of the MX Record's functionality, like being able to designate a backup server or multiple servers for a service is useful for other protocols as well.
	This is where the SRV Record comes into play
SRV Record is a general purpose MX Record that can work with any service.
The syntax of the SRV Record is a bit more complicated compared to the MX Record.
	The record specific data has four fields
	16-bit priority -> Works like the 16-bit preference field in MOX record
	16-bit weight -> Used for distribution of load (how traffic should be distributed among multiple servers that share the same priority value)
	Port -> Tells clients which port a service is running on
	Target -> Gives clients the domain name of a host offering the service
SRV Records also support owner names with an unusual format
![[Pasted image 20260620160201.png]]
The first two labels of the domain name are the **name of the service** and the **transport protocol** it runs over. Both with leading underscores. The rest is the domain name that the client actually specified. 
SRV-smart clients would look up SRC Records for a destination before falling back to A and Quad A Records
	If SRV records are found, it would be sorted based on priority(high to low, 0-highest and 65535-lowest)
	Then it would try to connect with the highest priority target on the port specified 
	If more than one SRV Record with the same priority is found, it would connect to the target in proportion to their weight fields
		If two records have the same priority and the same weight, the server would pick a record to go first randomly 
			If two records have the same priority but different weights, the record with the higher weight would net more of the traffic share
				Eg: If two servers A and B have equal priority but different weights of 80 and 20 respectively, server A would do the heavy lifting dealing with 80% of the traffic and server B would handle 20%.
Like with MX Records, clients must try all targets at a given priority before moving to ones with less priority
Not many clients support SRV Records.
	Computers that are members of an Active Directory Domain find their domain controllers by looking up SRV Records
	Minecraft clients find servers using SRV Records as well

#### TXT Record
Text records can be used to attach comments or almost anything else to domain names in the namespace
![[Pasted image 20260620162023.png]]
the type field is txt and the record specific data is in one or more double quotes. 
	technically the RDATA doesn't need to be double quoted but then it would interpreted as a series of double quotes
Any ASCII string can be safely used in the TXT Record RDATA. 
	Each string can be 255 bytes long and you can use multiple string to specify even longer data
	However it is best to stay under certain limits
		Text Records larger than 4Kb won't fit into most DNS based messages and a Text Message longer than 1300 bytes will be fragmented at the Ethernet Layer 

### Infrastructure Records: SOA Record, NS Record and Glue
Some records are used primarily by DNS itself and not by hosts or applications running on hosts. These are **SOA** and **NS Records**
SOA Records stands for Start Of Authority Records
	There is only one SOA Record in any zone and it is attached to the domain name of the zone
	The SOA records contains important zone parameters including the zone serial number
![[Pasted image 20260623180317.png|697]]
	Note: The last character of the first line of the record is an open parenthesis which tells the name server reading the record to ignore character turns and new lines from there to the matching closed parenthesis.
		These parenthesis can be used in any record but SOA Records are usually so long that you'll routinely see parenthesis in them
	The **MName** field, by convention, is the domain name of the primary name server of the zone 
		Some software that send dynamic updates to the name servers look at this field to help decide where to send those updates. However, if a zone is not dynamically updated, it doesn't matter what you put there
	The **RNAME** field is a special encoding of the email address of a person responsible for the zone.
		The format is different as the @ symbol is replaced with a dot(.)
			`Eg: dns@infoblox.com -> dns.infoblox.com`
		Only human beings and some spam miners use this field. However, it is considered polite to list a real address here so folks having trouble with your zone can contact you
	The serial number, refresh internal and expiration interval all have to do with transfers of the zone
		**Serial Numbers** are unsigned 32-bit values
			It acts as an indication of the version of the data in a zone and in incremented with each update which helps secondary name servers for the zone tell if it needs to do a transfer to update it's copy
			For **manually edited zones**, its common to use a format like YYYYMMDDVV (2026062401)
			It allows 100 updates a day
			For **automatically edited zones**(thru dynamic updates or rewritten via scripts), the serial number doesn't usually have a fixed format. Secondary name servers compare serial numbers using something called **sequence space** or **clock arithmetic**.
				In DNS, the calculation used for SIA serial numbers is called Serial Number Arithmetic and is also often called clock arithmetic because it treats serial numbers as a circular loop rather than a straight line.
				Since the serial number field is a 32-bit integer, the number can range from 0 to $2^{32} -1$ 
					DNS uses the half space rule. The total 32-bit space is $2^{32}$ and exactly half of that is $2^{31}$ 
						What this means is that we have $2^{32} -1$ from `0 - 4294967295` .
						 We have 4294967296 (including 0) namespace values for serial numbers and $2^{31}$ is half of the entire name space which is `2147483648` .
						 For a given serial number, the given $2^{31}$ values, even if they wrap around up to the maximum 32-bit number and default back to zero, are considered bigger and the previous $2^{31}$ values are considered smaller.
						 Thus, for the current serial number `s`, a new serial number `n` is considered newer if and only if
							 $(n - s) \pmod{2^{32}} < 2^{31}$ and n!=s 
	and is usually just incremented when the zone is changed so no special format is used. 
	The negative-caching TTL value controls how long a recursive name server may cache negative answers from the zone
		Negative responses include 
			NXDOMAIN (There is no such domain name)
			NODATA (There is no data of that in that type in the domain name)
		When negative responses are sent to the querier, SOA Records are also included in the query
#### Retry and Expire
If a refresh query fails(eg: a master fails to reply), the secondary servers wait for a **retry interval**, which is usually shorter than the refresh interval, before checking again. 
	The idea is that if a refresh fails, the secondary should try more aggressively.
	The secondary server retries as long as the expiration interval does not run out
**The Expiration Interval** is a timer and is the ultimate safety limit. It determines how long a secondary server can legally keep answering public internet queries using its local cache if the primary server doesn't respond. Every time a secondary server fails a refresh/retry check, the expiration timer continuously counts down towards zero unless the refresh/retry succeeds.
	If the expire interval passes and the secondary server is unable to determine if the zone data is up to date or not, it stops answering queries for data in the zone, figuring that the data it hold is now so old that it can't give it out safely. 
		The expire interval is usually many multiples of the refresh interval
	After the zone data expires, the secondary server begins responding to all queries to it with a error called **SERVFAIL** or "Server Failed". 

NS stands for **Name Server Records**
	NS Records list the authoritative name servers for a zone
	As with the SOA Record, the owner name must be the domain name of a zone while the record specific data list the domain name of a name server authoritative for that zone. 
		When an authoritative name server needs to tall a recursive name server which name servers are authoritative for a zone(like in a referral), an NS record is sent. 
![[Pasted image 20260627111156.png]]
	NS Records are always attached to the domain name of the zone.
	The record specific data is the domain name of a name server authoritative for that zone
		Since a zone can have more than one authoritative name server, they'll also have more than one NS record.
	NS Records stitch the namespace together
		The domain names of the root name servers are provided as NS records
		When an authoritative name server for a parent zone sends a recursive name server a referral to name servers authoritative some some sub-zones, it sends a list of NS records for that sub-zone. 
	Even **Root Hints** -> The list of name servers authoritative for the root zone are expressed as a set of NS Records along with their corresponding A and Quad A (AAAA) records.
	![[Pasted image 20260627180953.png]]
	Notice the NS record pointing to a.root-servers.net.
	![[Pasted image 20260627181156.png]]
	We can clearly see that the Root Hint contains both NS Records and Address Records
		This is because NS Records alone are not useful
			We know the name of the root name server but to query it, you need an address
		These accompanying A and Quad A Records are the glue that completes the delegation (glue records)
	Eg: A delegation from the com zone to the infoblox.com zone
	![[Pasted image 20260627182041.png]]
	This information alone is not enough to complete delegation
		Since we have no addresses, the recursive query would have to query these same infoblox.com name servers to find them. Not ideal. (Chicken and Egg Loop Thingy)
			- The internet root/TLD tells the recursive server "To find *example.com*, go ask its authoritative name server", which is *ns1.example.com*
			- The server asks the TLD to the IP address of *ns1.example.com*
			- The TLD tells the recursive server to look inside *example.com* which creates a loop
		Thus, to solve the problem, the parent zone contains both the A and Quad A Records for those name severs, even though normally, you'd find those records in the infoblox.com zone and not the com zone.
		Glue records are only necessary when the domain name, of the name server being delegated to, ends in the domain name of the zone being delegated to it.

#### WildCard Names
Wildcards aren't a type of record, but rather a type of label in a domain name
Instead of attaching a record to an explicit domain name, the leftmost or first label can be a wildcard `*` allowing a record to match queries for a while bunch of domain names.
Eg: 
`*.infoblox.com.  IN    CNAME infoblox.com.`
The wild card matches one or more labels in a domain name being looked up but only if a specific match isn't present in the zone
![[Pasted image 20260627195756.png]]
We have 
	an A record attached to the apex
	www owning a CNAME record
	test being delegated to ns1.test.infoblox.com
	finally a wildcard record 
		The wildcard record would matched the query for foo.infoblox.com or foo.bar.infoblox.com because they don't exist 
		The wildcard record however would not match the query for infoblox.com or www.infoblox.com, both of which clearly exist in the zone
			The wild card wouldn't also match the query for foo.test.infoblox.com since a delegation to *test* already exists.
				This is because when you create a standard wildcard record like 
				`*.example.com.  IN  A  192.0.2.5`, it acts like an instruction manual for the server to synthesize records only if two distinct conditions are met
					- The requested domain doesn't exist anywhere else in the zone file
					- The query type matches the wildcard record's type
						Since DNS matches queries by simultaneously checking the domain name and data type, should an MX record for `random.example.com` be queried, the server finds the `*.example.com` wildcard record but sees that it only contains an A record
							Since DNS is strictly separated by data type, an A record cannot satisfy an MX request. It will never automatically convert any record into another type. Thus, the server responds with `NOERROR`, which proves that the domain exists via the wildcard, and `NODATA`, which leaves the answer section empty
It is alright to think of wildcards as a default name in the zone
We can use wildcard records with many types of records including A Records, Quad A Records, MX Records and CNAME Records. 
Wildcards don't make much sense with SOA and NS records since they must be attached to a zone
	The wildcard will only match queries for the same type as the type the wildcard owns, except for wildcard CNAME records which will match any type and restart the query
		This happens because in DNS, CNAME is an alias, a pointer to another domain name. It simply points to another domain name. So when a query of any type is sent to the server and it hits the CNAME wildcard, it immediately redirects it to the domain name it is pointing to
			*"Whatever type your are looking for, you won't find it here. Go and look for that exact query over at `example.com` instead"*
Wildcards often change the responses for other query types
	Since in a zone with a wild card, all possible domain names seem to exist.
	Queries for non-existent domain types in a zone with a wildcard makes authoritative servers return `NODATA` instead of `NXDOMAIN`
When a query matches a wildcard, the authoritative name server synthesizes a new record with the domain name of the query with the data of the wildcard record. The wildcard itself is not returned.
	For the query *foo.bar.infoblox.com*, it would return
	`foo.bar.infoblox.com  IN  A  10.0.0.1`

#### RRset (Resource Record Set)
RRSet is a set of resource records with the same domain name, class and type.
	E.g: all the A records for www.example.com, all of its MX records, etc
Name servers may vary the order in which they return an RRset
BIND Name Servers support several configuable RRset orders
	- Round Robin/cyclic: Yje first record is rotated to the end in the next response
	![[Pasted image 20260627205139.png]]
	- Random: The records are returned in random order
	- Fixed: The records are always returned in the same order, which is the order in which they appear in the zone data
Some people use RRset order to distribute load among a set of servers but,
	The problem is that the name servers probably have no idea of the load on those servers or even if they're online or offline
	Intermediate name servers (eg: our local recursive name server) may reorder RRsets too.
In some case, the client may not use the first record returned
	If the client reorders, all bets are off 

#### Tracing Resolution with DIG
+norec means we are sending a non-recursive query to the server
![[Pasted image 20260701182543.png]]
- The authority section has 13 NS Records. This is delegation. This means that our query was delegated from the root name servers to the com name servers
- The additional section includes address records both IPv4 and IPv6 for each of these servers
![[Pasted image 20260701183110.png]]
- We have the 4 NS records pointing to the 4 Name Servers for infoblox.com `ns1,ns5,ns6,ns7`
- In the additional sections we have some address records both A and Quad A
Further following the NS we get
![[Pasted image 20260701183428.png]]
- We got a CNAME Record
- We can see that infoblox.com is an alias for the actual website name hosted on fastly
- Repeat this entire set of commands for the alias
![[Pasted image 20260701192703.png]]![[Pasted image 20260701192711.png]]

#### Stub Resolver Behaviour
This is the client side of DNS
It retransmits a query if a Name Server doesn't respond and will fall back from one name server to another if configured with multiple name servers.
	The details of how and when the resolver does this is dependent on their make and model
##### BIND Stud Resolver
The resolver in BIND is built in as a part of many UNIX and Linux-ish operative systems
You can generally query a BIND resolver to query upto three name servers
	For some operating systems, the limit is higher
The resolver works by querying the first server in the list and waits upto **5 seconds by default** for a reply
	If it doesn't get a response from it, it'll query another server in the list and wait 5 seconds 
If no response is received, the resolver doubles the waiting time and divides it by the number of name servers in the list and starts at the top of the list again with this new timeout value
Newer BIND resolvers only go through the same Name List twice
BIND Stub Resolvers max out at 3 by default
![[Pasted image 20260703162122.png]]
The aggregate time for a stub resolver to respond(timeout duration) is between 15s to 24s. While this may seem like a lot of time, name servers usually respond within a fraction of a second.

#### Windows Resolver
The windows resolver is a completely different type. It ships with windows OS.
However, there are different "species" of this software for different versions of windows, even in different service packs in the same version of Windows. 
The Windows Resolver is very aggressive, almost like a wild animal.
![[Pasted image 20260707151215.png]]
Notice, the timeouts are shorter than the BIND results. It also retries a total of four times
That is not the full account of its aggressiveness.

##### The Windows Retransmission Algorithm
On Unix and Linux-like systems, the entire OS has a single resolver configuration. 
Windows on the other had can have a different resolver configuration per network adapter 
	Each network may have a different list of name servers
On modern networks, many computers have **atleast** one wired and one wireless network adapter. 
Windows doesn't restrict itself to trying a single name server.
	It may shotgun queries to multiple name servers at once and it doesn't always run through the list of name servers from top to bottom either
	If a Name Server doesn't reply, windows will move it to the end of the list for that network adapter
 ![[Pasted image 20260710115822.png]]
	 - This is an image showcasing the configuration of a windows resolver in Windows 7
	 - This is the resolver configuration for the LAN adapter
		 - Usually that configuration is handled by the DHCP servers but those settings can be overridden here
		![[Pasted image 20260710121231.png]]
	- If we want finer grain control over the resolver configuration, we can configure them here
	![[Pasted image 20260710132620.png]]
Given all these per adapter list of name servers and the rotation of unresponsive name servers to the end of each list, the combined algorithm looks like follows
- Query the top name server on the preferred network adapter's lists and waits for the first time out (generally one second). 
- If no responses before the time out, it rotates that name server to the end of that list
- Query the top name server on all adapter's lists and then waits for the second time out
- If none of them respond, the resolver rotates the name servers to the end of their respective lists
- Query the top name server on all adapter's lists again and waits for the third time out
- If no response is provided, the resolver gets desperate and "shotguns" the queries to all of the name servers to all of the name servers on all of the adapter's lists simultaneously. It waits for the fourth timeout
- If no response is found, the shotgun process is done again
The good news is that the aggregate timeout is only 15 seconds in the worst case. 
The bad news is that if your DNS infrastructure starts slowing down, your windows resolvers are going to start shotgunning queries around, making recursive name servers resolve the same domain names in parallel which exacerbates the problem by slowing your name servers even further 

#### The Local Domain and The Search List
**~Contd. Domain Names~** 
In domain names, we know that domain names that end in a dot are absolute, written relative to the root. This is true for zone data files and for resolvers.
In a zone data file, domain names that do not end in a dot conveniently have the origin name appended to them
But how does a resolver complete a potentially incomplete domain name typed at the command line, in a web browser(address bar) or in a config file? 
	There are two mechanisms for doing this and which one is used depends on the context
##### Local Domain
A single domain name that's the domain that the resolver is in or the domain that it does most of its lookups in.
On Unix or Linux-like systems, the local domain name is used to interpret single level domain names in configuration files 
E.g: If you have *foo* on the hosts.equiv or .rhosts files, we obviously don't mean every host on the internet with a first label *foo*. We mean *foo* in our local domain. 
	assuming the resolver's local domain is foo.example
	"host1" is interpreted as host1.foo.example
`hosts.equiv -> A system file that lists users and hosts authorized to name r-commands(remote commands)`
`.rhosts -> User level equivalent to the /etc/hosts.equiv file. This file allows individual users to configure passwordless remote access to their specific account for the legacy r-commands-rsh,rlogin,rcp`
E.g2: Another example is if we type `ping my-server`, the OS appends our configured local domain name to the end of the query before sending it to the DNS server. Lets say our local domain name is `local`. The OS would append the query into `ping my-server.local`
##### Search List
A list of domain names appended to possibly incomplete domain names. 
The search list is often applied by network clients like SSH clients or web browsers to possibly incomplete names typed as arguments or in the address bar.
It is an **ordered list** of multiple domains that our device will systematically cycle through if a DNS lookup fails.
The elements of the search list is appended to the name **in order** and the result is looked up until an answer, usually and address, is found. 
Different resolvers have different rules as to when the argument is looked up as is.
Some resolvers may look up the domain name as-is first and if that doesn't return an answer, the resolver appends the first element of the search list and looks up the result. This continues until an answer is found or the list is exhausted.
	This process typically involves counting the number of dots in the argument and using that to decide whether the user actually typed the fully qualified domain name but neglected to add a trailing dot. 
Let us assume our /etc/resolve.conf file has the following list configured
```
search dev.lan prod.lan corporate.internal
```
If we attempt to connect to a machine named `database` (ssh database), our system will automatically perform DNS queries in this exact order
![[Pasted image 20260710154620.png]]
If none of these domains return an IP address, the resolver finally throws a "Host not found" error
Long search lists can result in unnecessary queries so it is a good idea to keep them short.

With the **Bind Resolver**, we can either set the local domain and make do with a default search list or set the search list explicitly. 
![[Pasted image 20260710155057.png]]
The `domain nxdomain.com` sets the local domain as nxdomain.com and sets the search list to the default. 
The `search inca.infoblox.com infoblox.com` explicitly sets the search list to those two domains in order and it also sets the default domain to `inca.infoblox.com`
It isn't good practice to list both domain and search lists. (Last One Wins approach -> The one listed last will be used domain or search)
The default search list includes just the local domain.
An explicit search list is limited to 6 domain names and 255 characters. If you're approaching that limit, there is a chance you're doing something wrong
The BIND resolver decides if to look up an argument as is before applying a search list by checking the number of dots in the name.
	By default, if the name has atleast one dot, it's looked up as is first before applying the search list. This threshold can be configured. 

The **Windows Resolver** sets its local domain to the same name as the Active Directory domain that it's part of. 
If it's not part of an Active Directory domain, we can set it manually or through DHCP or global policy. 
By default, the search list is derived from the local domain to include the local domain and all ancestor names with atleast two labels 
	The search list (Domain Suffix Search Order) is derived from the Active Directory domain's name by default, by "devolving" it
E.g: If the AD domain name was *ad.infoblox.com*, the default search list would be 
	- ad.infoblox.com
	- infoblox.com
Network Adapters can also have connection-specific domain names which are added to the search list
	These are called connection-specific DNS suffixes in Microsoft's terms

![[Pasted image 20260711095643.png]]
- The first field can be used to set the primary DNS suffix (Microsoft's terms for the local domain)
![[Pasted image 20260711095746.png]]
![[Pasted image 20260711095850.png]]
- Here we can see checkboxes that control what go into the search list. 
	- By default, both local and connection specific domains and well as the ancestors of the local domain
![[Pasted image 20260711100051.png]]
- This is where the connection specific domain is configured


#### Dynamic Update
##### History
Originally, the only way to update a zone data was to update a zone data file and then reloading the zone.
	This required admin intervention and stream editing the file, which was pretty risky.
		One mistake and the zone wouldn't load and your servers would start spewing zone fail errors.
As networks became more dynamic, the internet needed a way within the DNS protocol to add resource records to and delete record from a zone. 
in 1997, RFC 2136 was published. It was a codified mechanism to do this called **dynamic update**.
##### Update Messages 
Like in Queries and Responses, dynamic updates are carried in DNS messages and are identified with a unique OPCODE called *UPDATE* 
Updaters, which can be any devices with the appropriate code not necessarily a name server, send updates to the authoritative name servers for a zone to add records to delete from them. 
The sections of the DNS message after the header are repurposed as 
	- Zone
		- This section simply contains the SOA records of the zone to update
	- Prerequisite
		- This section contains an optional set of conditions which if included must be true in order for the subsequent update to be processed.
		- There are 5 types of prerequisites
			- RRset exists (value independant). E.g: www.infoblox.com has atleast an A record.
			- RRset exists (value dependant). E.g: www.infoblox.com has the A record 10.0.0.1
			-  RRset doesn't exist. E.g: www.infoblox.com has no A records.
			- Name is in use. E.g: www.infoblox.com owns atleast one resource record.
			- Name is not in use. E.g: www.infoblox.com doesn't own any resource records.
	- Update
		- There are 4 types of updates we can perform.
			- Add RRs to and RRset
			- Delete an RRset. E.g: delete all of www.infoblox.com's A records.
			- Delete all RRsets from a name. E.g: delete all records from www.infoblox.com
			- Delete a particular RR from an RRset. E.g: delete only the A record pointing to 10.0.0.1 from www.infoblox.com
			![[Pasted image 20260711111854.png]]
The updater is not required to set prerequisites, in which case, the update is processed, assuming the updater is authorized
The updater cannot send an update anywhere. Only a primary server for a zone can update that zone. Thus, updaters follow a process to try to ascertain which name server to send an update to. 
- First the enclosing zone for the domain name to be updated is found. This is done by sending an SOA query for the domain name they're trying to update. 
	- If the domain name is the name of a zone, it will return the zone's SOA record.
	- If the domain name is just the name in the zone, it will return a negative response but the authority section of that response should include the enclosing zone's SOA record
- Next, the NS records for the zone are looked up.
- The NS records are compared with the content of the SOA record's MNAME field.
	- If the MNAME matches one of the name servers in the NS records, that's probably the primary so it is placed at the top of the list of name servers to send the updates to. 
- The name servers for the NS records are added to the list 
- Then, it runs through the list of authoritative name servers trying to send updates to each one until you succeed. 
We can see that for dynamically updated zones, the convention is listing the primary in the MNAME field is pretty important.
What happens if a secondary listed in the NS records receives an update?
	Some secondaries have update forwarding enabled which means they will forward that update to their master server, otherwise, they will simply refuse the update.
A name server that is capable of processing an update by either updating the zone or forwarding the updates sends back a response code indicating a success. 

How is dynamic update used?
Windows computers can use dynamic updates to register their name to address mappings by dynamically adding A records. 
Statically addressed computers also use dynamic update to add pointer records to reverse map their addresses back to domain names.
	DHCP clients, however, will let a DHCP server add pointer records on their behalf
For non-Windows clients, some DHCP servers will use dynamic update to add both A and PTR records on it's clients' behalf. For example, the ISC DHCP server does this. In-fact it will add both A and PTR records even for Windows clients but we can configure it not to. 

##### Authorization of Updaters
Updaters may be authorized by IP address. However, given the act that updates are usually UDP based, that may not be very secure 
DNS also supports an authentication mechanism called TSIG(Transaction Signature). This is a cryptographic key shared by the updater and the primary name server processing that update. 
	Updaters can be authorized by TSIG key as well. Given a TSIG required a shared key, its useful for configuring one or a few DHCP servers and a primary server with shared keys. However, configuring a large set of updaters, say all your Windows computers with unique keys that they share with the primary is unwieldy 
	In order to address this issue, Microsoft came up with GSS-TSIG
GSS-TSIG uses the keys already stored in an Active Directory for every member computer to negotiate a shared secret between a member and a Microsoft DNS server. 
	Microsoft calls updates secured with GSS-TSIG as secure dynamic updates. 

#### Notify
**History**
With the advent of dynamic update, zone data could change more frequently than with manual editing. Computers could register themselves when they booted or received DHCP leases. However, secondary servers would only check for updates synchronously (when the refresh timer expired). This would induce a substantial weight before all of the authoritative name servers for a zone had dynamic new data
This setback inspired the creation of NOTIFY in 1996 (RFC 1996).
	It allows authoritative name servers for a zone to tell its secondaries when a zone data has changed. 
The primary send NOTIFY messages whenever it loads a new version of a zone or processes a dynamic update. 
	Secondary handle the receiving of a NOTIFY message from their master the same way they handle the refresh timer going off. 
		They query the master for the SOA record for the zone, then check the serial number to see if it incremented and if it has, they request zone transfers.
**Notifying Second Tier Secondaries**
To address situations in which secondaries get their zone transfers from other secondaries, secondaries send NOTIFY messages as well. 
All authoritative name servers for a zone send NOTIFY messages  to all the name servers in the list of NS records(except themselves obv) as well as any name server also listed in the MNAME field(since that is presumably the primary). 
The total effect like like a flood fill algorithm, the secondaries ignore NOTIFY messages from any name server but their configured master name server(s).
E.g
![[Pasted image 20260717081024.png]]
	- Here is an example of NOTIFY at work in a zone transfer dependency graph of secondaries transferring from other secondaries. 
	- We can see that the secondary NS at the bottom gets its zone transfer from the secondaries in the first tier (this assumes a valid primary is listed in the MNAME field)
	- The primary processes a dynamic update or reloads its zone data, it then sends a NOTIFY message to everybody in the list of NS records except itself.
![[Pasted image 20260717081637.png]]
	- The first three secondaries in the first tier accept the primary's notify message but the secondary in the second tier does not
![[Pasted image 20260717081812.png]]
	- This is because it is receiving a NOTIFY message from a name server other than its master
	- Once the first tier secondaries have finished processing their zone transfers, they'll send NOTIFY messages as well to every server in their list except the primary and themselves
![[Pasted image 20260717082040.png]]
	- In this example we can see that the first tier secondary name server in the left sends the NOTIFY message to the other three secondary name servers. However, this message is going to be refused because none of them use it as a master
![[Pasted image 20260717082239.png]]
	- The one in the middle then sends the NOTIFY message. The name servers in the left and right of it reject it because it is not their master but the secondary name server in the second tier accepts it since the message comes from its master
![[Pasted image 20260717082519.png]]
![[Pasted image 20260717082605.png]]
	-  The right most first tier secondary server sends its NOTIFY messages to the three name servers except itself and the primary which is rejected
![[Pasted image 20260717082706.png]]
![[Pasted image 20260717082731.png]]
	- NOTIFY is a top-down trigger. In this example, the secondary tier secondary server does not send the NOTIFY message to the name servers "above" it.

While all of these processes might look like extra traffic, NOTIFY messages are very small. Just a single UDP datagram so it need much juice.

#### Incremental Zone Transfer (IXFR)
**History**
Dynamic updates along with NOTIFY allowed the possibility of much more frequent zone transfers. Unfortunately, traditional zone transfers meant they carried an entire zone, even if you changed one single record
RFC 1995, codified in 1996, defined the Incremental Zone Transfer.
In IXFR, zone transfers can carry just the changes to the zone.
In IXFR, the secondary name servers send a query type of IXFR instead of AXFR, for the domain name of the zone and it includes the SOA record it holds for the zone in the query
	The master name server then determines what changed between that version of the zone and the current one. It verifies what was added and what was deleted.
	If those changes are larger than the zones or if it cannot determine the changes, it just sends the entire zone, otherwise it send only those changes (changes are also called **delta**).
	This process requires that the master name server keeps track of which changes to the zone correspond to which serial numbers(in the SOA record). 
In the best case, the delta is very small and may fit into a single UDP-based DNS message, which makes for very efficient zone transfers. 
	`Note: A delta being larger than a zone entails that the total combined network size(in bytes or packet overhead) of the step-by-step changes made to a zone since the last name server update is larger than the size of the entire zone database itself.`

#### Sender Policy Framework
SPF is an application of DNS that allows zone admins to combat the spoofing of emails. 
The standard email sending protocol of the internet is the SMTP or simple mail transfer protocol. 
	SMPT doesn't provide any way to authenticate the user of the email. 
	A sender can claim to be sending an email from a legitimate or well known domains only for their IP to be in random places uncommon to that domain. 
SPF allows zone admins to list the domain names or IP addresses of mail servers that are authorized to send mail from the domain names in the zone(domain name after the "@" symbol). 
	This data is added to text records, attached to the domain names that appear on the right side of email addresses. 
E.g:
![[Pasted image 20260717150300.png]]
	- The top text record is the one used by SPF
	- We can see that the Rdata begins with `v=spf1`, which distinguishes a text record meant for use by a SPF-smart mail servers, from other text records we might use. 
	- The +mx mechanism allows any mail servers listed in the MX records for the domain name to also send email addressed from the domain name 
		- In simple terms, when we add a +mx rule to our domain, we tell the internet to trust the servers that receive our mail and should an email be sent from that exact server, to accept it. 
	- In the example, we can see two more mx records; mx1 and mx2. These records allow infoblox.com addresses. 
	- The **+a** mechanism explicitly allows the host with the address that smtp.infoblox.com maps to, to send mail from infoblox.com addresses. 
	- The -all mechanism is a "Fail" directive that tells receiving email servers that any email claiming to be from `infoblox.com` that doesn't originate from the authorized servers listed above should be rejected.
	- Each mechanism begins with a "+" or a "-" 
		- "+" means that hosts matching the mechanism are authorized and "-" means that matching hosts are not authorized. 
	![[Pasted image 20260717192257.png]]
	![[Pasted image 20260717192333.png]]
	- all: Catch-All -> It is typically placed at the very end of an SPF record to dictate what should happen to any IP address that didn't match any of the preceding mechanisms (e.g., `-all` means hard fail for anyone else).
	- ip4: IPv4 Match -> Authorizes a specific IPv4 address or an entire IPv4 network range {in network range/prefix format} (using CIDR notation) to send emails on behalf of the domain.
	- ip6: IPv6 Match -> Authorizes a specific IPv6 address or an entire IPv6 network range
	- a: A Record Match -> The receiving mail server looks up the A record (IP Addy) for the specified domain. If the sender's IP matches that IP, it is authorized
	- mx: Mail Exchanger Match -> The remaining server checks the MX records (mail servers) for the specified domain. If the sender's IP matches the IP of one of those mail servers, it is authorized
	- ptr: (Pointer) Reverse DNS Match -> Checks if the sender's IP address resolves to a hostname (via PTR record) that ends in the specified domain name.
	- exists: Existence Check -> Performs a DNS lookup for the provided domain. It doesn't matter what IP address is returned; as long as any A record exists for that domain, the mechanism matches and authorizes the sender
	- include: Policy Delegation -> Tells the receiving server to also check the SPF rules of the specified domain. If that included domain's policy results in a "Pass", the original check passes. Useful when you have multiple domain names that use the same set of authorized senders. This is heavily used when relying on third party services like Google Workspace, Microsoft 365, or Mail Chimp, to send emails. 
	For Prefixes
	- + (Pass): The sender is authorized -> This is the default if no prefix is given
	- - (Fail): The sender is absolutely not authorized
	- ~ (Soft Fail): The sender is not authorized, but the email should be accepted and marked/flagged as suspicious
	- ? (Neutral): No policy can be stated; treat as if no SPF record exists

#### Who Supports SPF?
* Lots of software including Microsoft Exchange and Sendmail support SPF. -> May require patching to enable support
* The big internet providers also support it including Gmail, Yahoo Mail, Hotmail and Outlook

#### NSLOOKUP
* Standard DNS query tool, one of the oldest. 
* Used to send queries to name servers and print responses
* Honestly not a good tool as it is prone to printing misleading errors
	* Some versions at startup try to reverse map the IP address of the name server they're going to query to a domain name simply to print it in a banner
	* Some versions, if the reverse mapping fails won't even query that name server
	* Others will print ominous errors just because the reverse mapping fails, not because anything is wrong with the query you were intending to send
- Why learn it then? Because it is pervasive. Almost all versions of Windows and most versions of Linux and Unix feature NSLookup

![[Pasted image 20261116203918.png]]
![[Pasted image 20261116203928.png]]
This is a non authoritative answer - the answer comes from the cache of our local recursive server

##### NSLookups Types
![[Pasted image 20261116213559.png]]
![[Pasted image 20261116222029.png|621]]
	- querytype | type -> Sets the specific type of DNS record you want to lookup. By default its a for address
	- recurse -> Toggles recursion on or off. When on, if your local DNS server doesn't know the answer to your query, it will ask other auth servers across the internet on your behalf. When off, it will only reply with what it has locally. On by default
	- vc -> Forces nslookup to use TCP (Virtual Circuit) instead of the default UDP protocol for sending DNS queries. This is typically used when responses are too large to fit into a standard UDP packet (which causes truncation)
	- srchlist -> Sets or customizes the search list (ordered sequence of domain siffixes that nslookup will automatically append to short, single-component names when you perform a lookup)
	- search -> Controls whether the domain search list is actually applied when you type a short, single component hostname)like typing `server1` instead of `server1.example.com` . Turning it off (nosearch) prevents nslookup from appending those automatic suffixes.
	- timeout -> Sets the initial time (in seconds) that nslookup will wait for a response from the DNS server before retrying.
	- retry -> Specifies the number of times nslookup will resend a query if the DNS server fails to reply within the timeout period
	- debug | d2 -> Turns on detailed diagnostic and packet level debugging mode. `debug` (lvl 1) prints request/response packets and intermediate details, while `d2` (lvl 2) provides comprehensive, deep packet-level debugging information
	- ndots -> Sets the threshold for the number of dots (label separators like .) a domain name must contain before nslookup treats it as an absolute name (fully qualified domain name). If a name has fewer dots than the threshold, nslookup will first append your local search list domains before trying it as is. Default is 1 which means any name with fewer than 1 dot will have search list suffixes appended to it first.

##### NSLookup Non Interactive Mode 
NSLookup allows users to issue commands using different options by only using their unique first letter preceded by a dash (-).
```
nslookup -q=mx nxdomain.com.
```

The first field/option argument is the domain name you want to look up and the second name if you use one is the domain name or the IP Address of the server you want to query. 
	If you leave out the second argument, NSLookup will just use the resolver's configuration to figure out which name server to query

#### DIG
- Apparently its full form is Domain Information Groper.
- Another command line tool
- Many advantages to nslookup -> it prints all sections of the DNS message by default, including the header. 
	- It prints resource records in their presentation format by default. 
	- Along with many other cool features not present in NSLookup
- Isn't as pervasive as nslookup

##### Basic DIG commands
syntax:
```
dig [@server] name [type]
```
eg:
```
dig @10.0.0.1 soa nxdomain.com.
```
- You can mix up the format as you like because dig knows that you're specifying the server to query when you type in the `@` sign, followed by an IP address or a Domain Name. 
```
dig infoblox.com. mx @192.168.0.0
```
* There is almost no overlap between record types and legal domain names, except for TLDs like Mexico's MX.
* The default name server is the first one in your resolver's configuration and the default type is A
- Like nslookup, dig makes it easier to reverse map IP Addresses. You use the `-x` option followed by an IPv4 or IPv6 addy
```
dig -x 50.19.85.84
```
##### Some useful commands
![[Pasted image 20261117131359.png]]
![[Pasted image 20261117132225.png]]

- -t explicitly sets the query type. You might have to use this if you want to differentiate MX (the query type) from MX(Mexico's TLD)
- -q allows you to explicitly set the domain name to query . You don't always have to use this - just when you want to clear things up like if your command structure requires a clear separation of bugs
- -x performs a reverse DNS lookup (maps an IPv4 or IPv6 address back to its corresponding domain/ host name) by automatically formatting the address into the appropriate PTR format
- +tcp | +notcp tells digs to send TCP based queries (useful for zone transfers or large responses that get truncated) or UDP based queries. By default it sends UDP based queries
- +domain sets the default domain name suffix that will be appended to the queries if the specified name is a relative/short hostname
- +search | +nosearch controls whether dig applies the domain search list (defined in your resolver configuration) when you enter a short or relative hostname. +nosearch disabled it so only the exact name is provided is queried.
- +recurse | +norecurse toggles whether dig requests recursive resolution from the DNS server. 
	- +recurse tells the server to fetch the answer from other authoritative servers if it doesn't have it cached. 
	- +norecurse restricts the server to only return what it already knows locally or authoritatively.
- +short | +noshort: Toggles the output conciseness 
	- +short strips away all standard header. footer and status comments and prints only the core answer (like IP Addy) which is useful for scripts
- +time=T sets the timeout duration(in seconds) that dig wait wait for a response from the DNS server before timing out. 
- +tries=T sets how many total query attempts dig makes if previous requests times-out or fails
- ndots=T sets the threshold for the number of dots (.) a domain name must contain for dig to treat it as an absolute (fully qualified) name on the first try. If the name has fewer dots than T, dig will first append your system's search list suffixes before trying the raw name.

Tracing recursive requests across servers is very important when you want to troubleshoot a resolution failure. This function is built into dig
```
dig +trace www.infobloc.com
```











