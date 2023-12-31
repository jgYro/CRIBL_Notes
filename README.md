# Cribl Notes


## Cribl Stream

### Sources
- explore
- refine

### Pipelines
- route
- reduce
- collect
- transform

### Pipeline Terms
- Events: set of key value pairs describing something that occurred at a point in time
- Logs: system-generated record of data
- Metrics: Numeric representation of data measured over intervals of time
- Traces: Marks path taken by a transaction within an application to be completed
- Observability Lakes: Data lake is a system or repo of data stored in its natural/raw format as blobs/files

### What Type of Data?
- Any machine data
- Any source
- Schema agnostic

### How It Works
- Sources -> Custom Commands / Event Brokers / Fields (Metadata) -> Pre-Processing Pipelines -> Routes -> Post-Processing Pipelines -> Destinations

### Single Instance Deployment
- All features enabled
- All src and dst supported
- Small volume environemt
- No req for resiliency
- Lightweight processing
- Idea for test, dev, QA & eval purposes

### Distributed Deployment
- Managed centrally from leader node
- - Controls worker nodes
- Better compute

### Cribl Stream Terms
- Worker process: a process that handles data inputs, processing & output
- Leader node: an instance to centrally author configs and montiors deployment
- Worker node: a managed worker, whose configuartion is fully managed by the leader Node
- Worker group: collection of worker nodes that share same config
- Mapping ruleset: an ordered list of filters used to map workers to worker groups

## Cribl Install

### Stream Naming Best Practices
- Create 'cribl' user
- Run Cribl stream as user 'cribl' with all files owned by cribl
- In linux, cribl:cribl
- Use systemd or initd
- - Use boot-start command to enable cribl to start at boot time with systemd/initd
- In distributed system, Stream sets each Worker Node admin pw with random value, different from Leader admin pw

### Port Requirements
- Heartbeat/Metric: TCP: 4200 or TCP:9000
- Cribl UI:TCP:9000
- - If using curl command to stand up a worker from the leader, need to use 4200 as well as 9000 between the two
- - To customize ports, update /opt/cibl/default/cribl/cribl.yml

### Stream Git Requirements
- Stream requires to Git >=1.8.3.1 on the host where the leader node runs
- Config changes must be committed to Git before they're deployed
- Leader Node uses Git to:
- - Manage config across work groups
- - Provide users an audit trail of config changes
- - Allows users to display diffs between current and prev configs

### Cribl Stream install

1. Add User Cribl: ```sudo adduser cribl```
2. Navigate to Cribl dir: ```cd /opt```
3. Download stream from Cribl.io: ```sudo curl -Lso - $(curl -s https://cdn.cribl.io/dl/latest) | sudo tar zxvf -```
4. Change ownership of Cribl: ```sudo chown -R cribl:cribl cribl```
5. Move to cribl/bin: ```cd cribl/bin```
6. Enable boot start services: ```sudo ./cribl boot-start enable -u cribl```
7. Start Cribl: ```sudo systemctl start cribl```
8. Check Cribl status: ```sudo ./cribl status```
9. Access UI: ```http://xxx.xxx.xxx.xxx:9000```


## Basic Distrbuted Settings

### Set Instance
- Leader
- - Need to install additional worker nodes
- - - Log in to worker node
- - - ```curl http://<leader hostname or ip>:9000/init/install-worker.sh?token=<token> | sudo sh -```
- - - Repeat for all worker nodes
- Managed Worker
- Single Instance

## Sources


### Types
- Push-based: send data to stream
- - Normally send to a set of Stream Workers through a load balancer
- - Some sources with native load-balancing capabilities and shoul dbe pointed directly at Stream nodes
- Pull-based: fetches data from

### Collectors
- Enables fetching from data from local or remote sources
- Basic colelction process
- - Worker Node receives config needed to execute collection job
- - Collection job is made of one or more tasks:
- - - DISCOVER the data to be fected
- - - FETCH data that matches filters
- - - Pass the results to a Route or into speciic pipeline & destination
- Types
- - Filesystem/NFS: Collecting data from a local or a remote filesystem location
- - S3: Colelcting data from Amazon S3 stores
- - Script: Fliexble data collection configured via custom scripts
- - REST: Collecting data from REST endpoints, supports multi DISCOVER & Collect options
- - Database: Coleclting data from MySQL/SQL/Postgress
- Define Process Settings
- - On-demand (ad hoc) or scheduled colelction
- - Filter expression to match the data against, the time range, etc
- - Run Mode: Preview, Discovery or Full Run

## Destinations

### Types
- Streaming: Accept events in real time/mini-batches
- Non-streaming: Accept events in (large) groups or batches
- - Staging location: A local fileystem, used to batch events into files
- - When condition is met: Staged files are moved to their destination
- - Conditions: File max size; File max open time; File max idle time
- - Maximum conditions: If a new file needs to be open, max conditions are enforced by closing older fiels in the order in which they were opened
- Output Routers: grouping of meta-destinations
- - Meta-destinations that allow for rule-based (real) destination selection
- - Rules are evaluated in order, top>down, with the first match being the winner
- - An Output Router cannot reference another
- - Events that do not match any of the rules are dropped
- - Data can be cloned by setting the Final flag to No

### Data Delivery
- Data is delivered to destinations >'at-least-once' basic
- If a dst is unreachable, Stream can be configured with these Backpressure behavior options:
- - Block: Block new incoming events
- - Drop: Drop events addressed to that Destination
- - Queue: Queue events to that Destination (Persistent Queuing)
- Default behavior is Block

## Routes

### Types
- QuickConnect: Visually connect Stream Sources to output Destination via drag-n-drop
- Routes: Completely configure data through Stream by defining a series of filter expressions to process the event
- - Direct data to Pipelines
- - Evaluate incoming events against filters
- - Each Route can be associated with ony one Pipeline and one output
- - Routes default with 'Final flag' set to Yes
- - Evaluated in order
- - List of filters used to find a matching pipeline and a destination to send events to:
- - - Filters: js expressions (truthy/falsy)
- - - Routes are evaluated in order: Top > Down
- - Final Toggle:
- - - Yes: matching events with by consumed by Route & not evaluated against any other Routes below it
- - - No: clones of matching events will be processed by the configured pipeline. The original events will continue down the rest of the Routes
- - Can accept data from multiple sources, but each route can be associated with only one pipeline & destination

### Route Strategies
- Most-specific first or most-general first
- Minimize the number of filters/Routes an event gets evaluated against
- Examples:
- - If cloning is not needed at all, start with the broadest expression at the top, to consume as many events as early as possible
- - If cloning is needed on narrow set of events, do that upfront, then follow it with a Route that consumes those clones immediately after

### Route Groups
- Routes can be moved up and down the Route stack together
- UI only: Routes in a group still maintain their global position order

## Pipelines
- List of functions that process events
- Events always move in the direction that points outside of the system
- Functions are evaluated in order: Top > Down
- Different pipeline "types"
- - Pre-processing
- - Post-processing
- - Main-processing

### Functions
- Building blocks of pipelines
- Discrete processing on an event
- Javascript
- Work only on events that match their filter condition
- Final Toggle:
- - No: Pass resulting events down
- - Yes: Short-circuit functions below

| Description | Function Names |
|---|---|
|Add, remove, update fields|Eval, lookup|
|Find & replace, sed-like, obfuscate, redact, hash|Mask|
|Add GeoIP info|GeoIP|
|Extract Fields|Regex Extract, Parser|
|Extract Timestamps|Auto Timestamp|
|Drop Events|Drop, Regex Filter, Sampling, Suppress, Dynamic Sampling|
|Sample events (e.g., high-volume, low-value data)|Sampling, Dynamic Sampling|
|Suppress events (e.g., duplicates)|Suppress|
|Serialize / change format (e.g., convert JSON to CSV)|Serialize|
|Convert JSON arrays or XML elements into own events|Unroll, JSON Unroll, XML Unroll|
|Flatten nested structures (e.g., nested JSON)|Flatten|
|Aggregate events in real-time (i.e., statistical aggregations)|Aggregate|
|Convert events in metrics format|Publish Metrics|

## Cribl Stream Packs
- Pre-built configurations: simplify the deployments & use of Stream product
- Improve time to value: provides out of box configs
- Enable plug & play dpeloyments for specific use cases
- Allows medium/large deployments sharing configs & content across multiple worker groups
- https://packs.cribl.io
- Allows ability to quickly allow troubleshooting by ease of replicating setup
- Work at worker group level in distributed deployment

## Processing

### Event Model
- Events: key-value pairs
- Fields that start with a double underscore are Cribl internal fields "__inputId"
- If an event cannot be JSON-parsed, all of its content will be assigned to field "_raw"
- If timestamp is not configured, the current time in UNIX epoch will be assigned to "_time"

### Processing Order
1. Sources: Data arrives from external providers
2. Custom command processor: External command consumes the data via stdin, processes it, and sends its output via stdout
3. Event breakers: Break up incoming bytestreams into discrete events
4. Fields/metadata: Add fields to each incoming event; Similar to Eval function
5. Input conditioning pipeline: Pre-prcess (normalize) data from this input before it reaches the Routes
6. Routes: Map incoming events to Processing Pipelines and Destinations
7. Processing pipelines: Events transformations via series of Fuctions
8. Output conditioning pipeline: Post-prcess (normalize) data to each destination
9. Destinations: Each Route/pipeline combination forwards processed data out to a destination

### Javascript Expressions
- JS Expressions are valid units of code that resolve to a value
- Every syntactically valid expression resolves to some value, but conceptually, there are two types of expressions:
- - Assign value to a variable: 
- - - ```x = 42```
- - - ```newFoo = foo.slice(30)```
- - Evaluate to a value
- - - ```(Math.random() * 42)```
- - - ```3 + 4```
- - - ```'foobar'```
- - - ```42```

### Javascript Comparison Expressions
- By default, Javascript evaluate operators form left to right, in order of precedence
- ```!``` not
- ```&&``` and
- ```||``` or
- ```==``` (equal value)
- - ```5 == 5 ```(true)
- - ```5 == 4 ```(false)
- ```===``` (equal value AND equal type)
- - ```5 == 5 ```(true)
- - ```5 == "5" ```(false)
- ```!=``` (not equal)
- - ```5 != 4 ```(true)
- - ```5 != 5 ```(false)
- ```!==``` (not equal value AND equal type)
- - ```5 !== 4 ```(true)
- - ```5 !== "4" ```(false)
- ```>``` (greater than)
- - ```5 > 4 ```(true)
- - ```4 > 5 ```(false)
- ```<``` (less than)
- - ```4 < 5 ```(false)
- - ```5 < 4 ```(true)
- ```>=``` (greater than or equal)
- - ```5 >= 4 ```(true)
- - ```5 >= 5 ```(true)
- ```<=``` (less than or equal)
- - ```4 <= 5 ```(true)
- - ```4 <= 4 ```(true)

### Javascript Methods
- ```.startsWith```
- - Returns ```true``` if a string starts with the specified string
- ```.endsWith```
- - Returns ```true``` if a string ends with the specified string
- ```.includes```
- - Returns ```true``` if a string contains with the specified string
- ```.match```
- - Returns an ```array``` containing the results if the string matches with a regex
- ```.indexOf```
- - Returns the position of the first occurrence of the substring

### Filters
- Used in:
- - Routes: to select a stream of the data flow
- - Functions: to scope or narrow down the applicabilty of the function
- Filters are expressions that must evaluate to either true (or truthy) or false (or falsy)
- Example:
- - ```source.endsWith('.log') || type=='vpcflow'```

|Truthy|Falsy|
|---|---|
|true|false|
|42|null|
|-42|undefined|
|3.14|0|
|"foo"|Nan|
|Infinity|''|
|-Infinity|""|

### Value Expressions
- Values expressions are typically used in Functions to assign a value to a new field
- Examples:
- - ```hour = Math.floor(_time/3600)```
- - ```index = host.endsWith('.internal') ? '_internal': 'main'```
- - ```location = 'New York, NY'```

## Data Samples
- Samples are files containing a set of events that can be replayed and previewed while authoring pipelines and configurations:
- - Uploaded (attached)
- - Pasted
- - Captured
- - - Can have live captures after any pipelines
- - - 4 capture points, save captures as Sample files and replay later as needed

### Datagens
- Enables users to generate sample data to troubleshoot
- Datagen template files ship with Stream
- Tempaltes can be created from a sample or capture
- Datagens are managed similarly to data sources

### Replay
- cribl steam deployment between log source and analysis system allows:
- - Data to be archived in a cheap storage mechanism for retention
- - Ability to "reply it"
- Simultaneously routes a full fidelity copy of the data to low-cost object storage
- Can point to data store and replay it back
- Received data can then be routed to an existing or new analytics system or system

## Knowledge Resources

| Resource | Description |
|---|---|
|Lookups|Data tables Stream uses to enrich events as they are processed by the Lookup Function|
|Event Breaker Rules|Rulesets are collections of Event Breaker rules that are associated with Sources|
|Parsers|Definitions and configurations leveraged by the Parser Function|
|Global Variables|Reusable JavaScript expressions that can be accessed in Functions in any Pipeline|
|Regexes|Regex Library that continas a set of pre-built common regex patterns|
|Grok Patterns|Grok Patterns library that contains a set of pre-built common patterns, organized as files|
|Schemas|JSON definitions that are used to match (validate) JSON events|

#### Lookups
- Enrich with lookup table or csv
- Allows to look up from field in data with a field in another csv
- Supported Formats:
- - CSV, Gzip, mmdb
- Adding Files:
- - Upload
- - Create

#### Event Breaker Rulesset
- Break incoming data into multiple events
- Event breaker types
- - Regex, File header, JSON array, JSON New line, Delimited, Timestamp, CSV

#### Parsers
- Extract fields from events or reserialize
- Parser Types
- - CSV, Common log format, Ext log file format, key-value pairs, JSON object, Delimited value

#### Global Variables
- Resusable javascript expressions

#### Regex Library
- Repository of regex patterns; SSN, Zip code, email, etc

#### Grok Patterns
- Advanced Grok patterns

#### Schemas
- Validating JSON
- rules based on validation

#### Parquet Schemas
- Used for large data, performant
- Good with compression

#### Database Connections
- MySQL
- SQL
- Postgres

## Admin Course

### Git

|Git Command|function|
|---|---|
|git init|start or re-initialize a repo|
|git add|stages the current state of files|
|git commit|bundles changes > moves them into HEAD|
|git push/pull|syncs local and remote repos|
|git diff|differences between files in repo and files in cwd|
|git status|files that have changed, been added or are untracked|
|git stash|"stash" local changes|
|git log|history of commits|
|git rm/mv|deletes files from or move files around repo|
|git checkout|switch to a branch, pull fiels from other branches into working directory|

### Git in Stream
- Creates local git repo in of the following locations
- - $CRIBL_VOLUME_DIR
- - Cribl Stream Install directory($CRIBL_HOME)
- Commit button executes:
- - ```git add``` on changed files
- - ```git commit``` to sync changes to the local repo (HEAD)
- ```git log``` provides full view of changes to with Cribl Stream audit log
- Stream can push and pull changes to/from remote repo
- Requires Enterprise License
- Connect each Stream instance to a different branch in the same repo

### Architecture Options
- Deployment Options
- - Singles instances
- - Worker processes
- - Distributed deployment
- Stream Components
- - Leaders 
- - Workers
- - Worker Groups
- Cloud Deployments
- - SaaS
- - Hybrid

### Leader High Availability
- If primary leaders fials, becomes unresponse:
- - Leader HA selects new leader
- - All current state and metrics stored
- - Default Cribl.Cloud service

### Stream Leader
- In distrbuted Deployment
- Gui & REST access
- Authentication & enforcement of RBAC
- Worker configuration - pull-based only
- Work Queue (discovery & collection jobs)
- Holding State (certain pull sources)
- Supports Disaster Recovery services

### Stream Workers
- Pulls configuration from Leader
- Reports metrics to Leader
- Receives from push-based source
- Collects from pull-based source
- Colector jobs
- Pushes results to destinations
- Handles all data processing
- HA discussion

### Workers & Worker Groups
- Worker Process: A process within a Single Instance, or within Worker Nodes, that handles data inputs, processing and output
- Worker Node: An instance running as a managed worker, whose configuration is fully managed by the Leader Node
- Worker Group: A colelction of Worker Nodes that share the same configuration

### Distributed Deployment Architecture
- Leader Node: An instance running in Leader mode, used to centrally author configurations and monitor a distrbuted deployment
- Mapping Ruleset: An ordered list of Filters, used to map Workers to Worker Groups

### Topologies
- One Leader, One Worker Group
- - Small deployment
- - Serving single department
- - Spanning single location
- One Leader, Multiple Worker Group
- - Supports larger deployment
- - Serving mulitple departments
- - Spanning mulitple locations
- - Enable a global footprint
- Worker Group to Worker Group
- - Compression, up to 90 percent
- - Secure Communication
- - Consolidated View
- Configuration: Remote Data Center
- - Source: Key value pairs
- - Route: Key value pairs to JSON
- - Destination: TCP JSON / Auth Token / Compression
- Configuration: HQ Data Center
- - Source: TCP JSON / Auth Token
- - Route: Any Data Shaping Required
- - Destination: Splunk

### Worker Groups
- A set of worker nodes that share the same configuration
- Organization with on-prem workloads plus cloud workers:
- - Create data center worker group and a cloud worker group
- Organization with multiple data centers:
- - Set worker groups with unique configurations per location
- Organization with widely variable workloads:
- - Dedicated worker groups for different sources

### Worker Processes
- Worker processes operate in parallel
- Data in a single connection will be handled by a single worker process
- Data should over multiple connections
- - It's better to have 5 connections to TCP 514, each bringing in 200gb/day, than one connection at 1tb/day
- Each worker process will maintain & manage its own outputs
- - If an instance with 4 worker processes is configured with a Splunk output, the the Splunk destination will see 4 inbound connections

### Worker Mappings: Rule Sets
- Map workers to worker groups
- List of rules that evaluate filter expressions on workers' heartbeat payloads
- Order matters > filter supports full js expressions
- Ruleset matching strategy is first-match
- One worker to one worker group
- At least one worker group should be defined & present in the system
- Only one mapping ruleset can be active at any one time

### Worker Process: Load Balancing
- Uses Node.js cluster module
- Uses round-robin process
- Stateless, if one fails, sends to next available process
- Worker node spans worker processes distributes incoming TCP connections
- Load Balance out to 120 indexers
- - Uses Time & Volume

### What Affects Performance Sizing
- Performance
- - Event Breaker Ruleset
- - Number of Routes
- - Number of Pipelines
- - Number of Clones
- - Health of destinations
- - Persistent Queueing

### Sizing by Processor
- Cribl guidelines assume that 1 physical core is equivalent to:
- - 2 Virtual/hyperthread CPUs (cCPUs) on Intel/Xeon or AMD processorZ
- - - vCPU vs CPU vs Core: hyperthreading
- - - 200 gb/vCPU overall throughput - at 3.0 ghz
- - - Sustained CPU rate vs burst rate
- - 1 (higher-throughput) vCPU on Graviton2/Arm64 processors
- - - Same general considerations as intel
- - - Throughput in excess of 200gb per core overall
- - - ARM is generally less expensive
- Allocate 1 physical core for each 4--gb/day of in+out throughput
- - Sum is estimated input & output volume, divide by 400gb
- - 100gb in -> 100gb out to each of 3 destinations = 400gb total == 1 core
- - 3tb in -> 1tb out = 4tb = 10 physical cores
- - 4tb in -> full 4tb to destination A, 2tb to destination b = 10tb total = 25 physical cores

### Leader Node Sizing: Recommended
- OS: Linux: RedHat, CentOS, Ubuntu, AWS Linux, Suse (64bit)
- System: CPU 4 physical cores (1 physical core assumed equivalent to 2 virtual cpu's), Ram 8gb, disk 5gb free
- Tasks:
- - Configuration authroing & tracking 
- - Monitoring & dashboards
- - Diganostics/log search
- - Scheduling & state tracking

### Worker Node Sizing: Recommended
- OS: Linux: RedHat, CentOS, Ubuntu, AWS Linux, Suse (64bit)
- System: CPU 8 physical cores (1 physical core assumed equivalent to 2 virtual cpu's), Ram 32gb (min 8gb), disk 5gb free
- Tasks:
- - Handle data ingress
- - Handle data processing
- - Handle data egress

### Remote Repo
- System Down
- Install Git on Backup Node
- Recover config from remote repo
- Restart Leader Node
- Back operational

### Recover Process: Commands
- Install Cribl
- git init
- git remote add origin <git repo .git>
- git fetch origin
- git reset --hard origin/leader
- git show --abrev-commit
- start Stream Leader

### Stream CLI Basics
|Command|Function|
|---|---|
|```./cribl help -a```|Show help information|
|```./cribl start```|Start Cribl Stream|
|```./cribl stop```|stop Cribl Stream|
|```./cribl restart```|restart Cribl Stream|
|```./cribl status```|Show Cribl Stream status|
|```./cribl mode-edge```|Configures Cribl Edge as a single single-instance deployment|
|```./cribl mode-managed-edge```|Configures Cribl Edge as an Edge Node|
|```./cribl mode-master```|Configures Cribl Stream as a Leader instance|
|```./cribl mode-single```|Configures Cribl Stream as a single-instance deployment|
|```./cribl mode-worker```|Configures Cribl Stream as a worker instance|
|```./cribl pack <sub-command> <options> <args>```|Manage Cribl Packs|
|```./cribl diag <sub-command> <options> <args>```|Manages diagnostic bundles|
|```./cribl diag create -d```|Create Cribl Stream diagnostic bundle in debug mode|
|```./cribl diag diag send -o```|Send Cribl Stream diagnostic bundle to Cribl Support|
|```./cribl diag list```|List existing Cribl Stream/Edge diagnostic bundles|

### Diagnostic Bundle Contents
- Folders:
- - default: Contians all the aPacks installed as well as .yml config files
- - local: Same as default but contains the changes made to Cribl Stream
- - log: Contains log files including access.log, audit.log, cribl.log, notifications.log, ui-access.log, etc
- - state: Contains the state of the system
- - system.json: Contains information about the state of the system running Cribl Stream
- - groups: This directory will appear if you ahve worker groups configured
- - - This folder will have the same folder strcture as above since it is a diag for each worker group

### Key Settings: Pull Sources
- Poll Interval: Specifies in min how often to scrape target by metrics
- Leader node required: Determines whether leader node is required
- Discovery Phase: When worker nodes discover what info is available
- Collection Phase: Collection jobs may be paginated, so multiple jobs may occur

### Additional Source Details
- Source Persistent Queuing
- - Implemented on outbound side, meaning each source can take advantage of a destination's queue
- - Some push sources have build-in buffer to handle Cribl Stream Backpressure
- - - Without buffering > Cribl SPQ required
- - - With limited in memory buffering > Cribl SPQ recommended
- - - With ability to buffer on Disk > Cribl SPQ not needed
- - TLS
- - - A TLS connection is when certs, keys CA chains and cipher suites all come together

### Stream Sources: Balancing Push
- Data pushed to Stream will need to be balanced across the worker nodes
- Some senders have load balancing built-in
- Open protocols will usually require a load balancing appliance/application
- - Syslog
- - HEC

### Stream Sources: No new agents
- Re-using already installed agents removes key barrier to entry

### RBAC
- Roles: Logical entities that are associated with one or multiple policies (groups of permissions)
- - Reader all role: For read-only/auditor type role created by default as needed
- Policies: A set of permissions that provide access or perform an action on a specific Cribl Stream object or objects
- Permissions: Access rights to navigate to, change or delete
- Users: Roles are mapped to Cribl stream users


### Stream Sources: Live Capture
- All sources can be captured live "off the wire" to serve as validation of delivery and testing rulesets
- Live feedback on changes before deployment
- Capture any samples at any point in the process

### Collectors
- Stream Colelctors that are designed to ingest data intermittently rather than continuously
- Collector can be ran scheduled or ad-hoc basis
- Support the following data types:
- - Azure Blob, Google Cloud Storage, REST, S3, Splunk Search, Health Check, Database, File System, Script

### Single Deployment Worker Node Collector Job
- Prepare the infrastructure to execute a collection job
- Discovers the data to be fetched
- Fetches the data that match the run filter
- Passes the results either through the Routes or into a specific pipeline

### Distributed Deployment Worker Node Collector Job
- Collectors are configured per worker group
- Leader node kicks off job for worker group
- Worker node execute the tasks to its entirety
- Leader node oversees the task distribution and tries to maintain balance across jobs
- Cribl Stream uses "Least-In-Flight Scheduling"
- Suppression or aggregation not possible due to "Share nothing" structure of processes
- - Need redis/similar app to perform stateful suppresion and stateful aggregation

### Stream to Stream
- Helps with slow connections
- Compression
- Encrypted via TCP JSON

### Cribl Expression Methods

|Cribl Expression|Function|
|---|---|
|C.Crypto|Data Encryptoin and Decryptoin Functions|
|C.Decode|Data decoding functions|
|C.Encode|Data encoding functions|
|C.env|Environment functions|
|C.Lookup|Inline lookup functions|
|C.Mask|Data masking functions|
|C.Misc|Miscellaneous Utility fucntions|
|C.Net|Network functions|
|C.Os|System Functions|
|C.Schema|Schema functions|
|C.Secret|Secrets-management functions|
|C.Text|Text functions|
|C.vars|Global variables|
|C.version|Cribl Stream version|

### TLS
1. A private key created via openssl or similar
2. Using a priv key, a public key is created and embedded in a Certificate Signing Request(CSR)
3. The CSR is signed, either by it's own key or a CA's key
4. The cert now has a priv key

- A cert cannot exist without being signed
- Public key (in signed cert) can encrypt/verify data
- Private key can decrypt/sign data
- Caveat: Entity posessing the priv key may not be the rightful owner

### Certificate Authorities (CAs)
- CAs are used to sign Cert Signing Requests
- Public vs Private - depends on need
- The first/top-level CA is the root -> assertion of trust
- The second CA is a subordinate/intermediate - optional but best practice

### Self-Signed Certificates
- Signed by entity whose identity it certifies
- Every root CA cert is self-signed
- Every self-signed cert is also a root but not necessarily a CA
- Still provides confidientiality, but authenticity and data integrity are suspect
- CA-signed certs mitigates these issues
- One step further is having the CA root cert deemed a trusted root by applications

### Level of trust
- Self-signed certs
- Private CA-signed certs
- Public CA-signed certs
- CA-signed certs whereby the CA is deemed trusted

### Certificate Chains
- Chains exist when a non-self-signed certificate is involved
- Many public CAs use chains to protect their root certs
- Frequently used within organizations handling their own signing
- Validating chains - start at the bottom and moves up the chain to the root

### KMS (Key Management Service) Overview
- Cribl Stream encrypts secrets stored on disk
- The keys used for encryption (cribl.secret) are managed by KMS
- The keys are unique to each WOrker Group + Leader
- Encryption key can be managed by Cribl Stream or by an external KMS
- Secrets encrypted by the key:
- - Sensitive information stored in configs
- - Data encryption keys stored as configs

### Stream Cert Validations
- Configuring settigns as a TLS server:
- - Authenticate Client (mutual auth) - if true, server requests client cert.
- - - Off by default
- - Validate Client - Clients whose certs aren't authorized have connections denied
- - Mutual auth enables optional CN validation via regex
- Leaf cert expiration and validation of CA chain then:
- - CN / SAN checks RFCs
- - - Only one is checked, regardless of no matches. SAN checked first, if values exist
- - - IPs are only accepted if they are in both SAN and Subject Attributes
- Stream as a client can validate the remote server cert using "Validate server certs" toggle
- Some destinations allow rejecting unauthorized (self-signed certs)
- If GUI does not provide a Reject Unauthoried toggle, then a global one can be used
- - Requires a restart
- - Must be included in systemd unit file

### Trace
- Traces represent the end to end request flow through a distributed system
- The data structure of traces look almost like an event log
- Traces are made up of spans
- - traceid
- - name
- - id

### Leader Node Logs
- API/main process in $CRIBL_HOME/log/ directory

|Logfile name|Description|Equivalent on LOGS PAGE|
|---|---|---|
|```cribl.log```|Principal log in Cribl Stream. Includes telemetry/license-validation logs. Corresponds to top-level ```cribl.log``` on Diag page|```Leader > API Process```|
|```access.log```|API calls, e.g., ```GET /api/v1/version/info```|```Leader > Access```|
|```audit.log```|Actions pertaining to files e.g., ```create```, ```update```, ```commit```, ```deploy```, ```delete```|```Leader > Audit```|
|```notifications.log```|Messages that appear in the notification list in the UI|```Leader > Notifications```|
|```ui-acess.log```|Interactions with different UI components described as URLs, e.g., ```/settings/apidocs```, ```/dashboard/logs```|```Leader > Notifications```|

- Config Helper process in $CRIBL_HOME/log/group/GROUPNAME directory

|Logfile name|Description|Equivalent on LOGS PAGE|
|---|---|---|
|```cribl.log```|Messages about config maintenance, previews, etc|```GROUPNAME > Config helper```|


- API process in $CRIBL_HOME/log/ directory

|Logfile name|Description|Equivalent on LOGS PAGE|
|---|---|---|
|```cribl.log```|Messages about the worker communicating with the Leader node and other API requests e.g., sending metrics, reaping job artifacts|```N/A```|


- Worker Process in $CRIBL_HOME/log/worker/[wp#]/

|Logfile name|Description|Equivalent on LOGS PAGE|
|---|---|---|
|```cribl.log```|Messages about the worker processing data|```N/A```|

### Upgrade Sequence
- Single Instance
- - Upgrade the instance
- Distributed Deployment
- - Upgrade the leader
- - Upgrade the workers
- - commit & deploy

### Prearing for an upgrade
- Default files will be overwritten
- Customs functions: move to ```$CRIBL_HOME/local/cribl/functions/```

### Manual Upgrade Steps for Single Instance
- Stop Stream
- Back up $CRIBL_HOME
- Uncompress new version over the old one
- Start Stream
- Validate your Stream environment

### Manual Upgrade Steps for Distributed Deployment
- Commit & deploy
- Ugprade the Leader Node (Worker nodes can continue running)
- - Stop Stream
- - Back up $CRIBL_HOME
- - Uncompress new version over the old one
- - Start Stream
- - Validate your Stream environment
- Upgrade the Worker Node
- - Wait for all the workers to report the leader
- - Stop
- - Uncompress new version over the old one
- - Start Stream
- Commit new software version changes
- - Ensure that all workers have reported with new version
- - Commit & deploy after verifying all workers are upgrade

### GitOps
- Need dev server and prod
- dev branch and prod branch
- Can specify whether or not prod/dev data can be sent
1. Make changes in Dev system
2. Commit & push changes to remote repo
3. Create pull request
4. Merge Pull request
5. CI/CD makes api to cribl stream

## Top Challenges

### Privilege Ports
- Do not run Cribl as root
- Cannot access ports 1-1024
- On a POSIX linux system run the following:
- - ```setcap cap_net_bind_service=+ep $CRIBL_HOME/bin/cribl```
- - Will need ran on every ugprade

### Too many open files
- Do not used high cardinality fields in the expression
- Configure max open files option
- LimitNoFILE=2048 in SystemD file

### OOM out of memory errors
- shown in cribl_stderr.log, gets overwritten on system startups
- Memory limits can be configured

### Cloning Workers
- Do not duplicate GUID, as they should be unique
- Remove .dat file in ```$CRIBL_HOME/local/cribl/auth```
- Regenerated on each run

### Resetting Lost Password
- Cribl.secdret file is located at ```$CRIBL_HOME/local/cribl/auth/cribl.secret```
- To manualy add, change or restore a pw, replace affected user's passwd k/v pair with a password key in the following format:
- - ```"password":"<newPlaintet>"```

### Pipeline Profiling
- Review metrics and problematic functions
