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

### Cribl Stream Packs
- Pre-built configurations: simplify the deployments & use of Stream product
- Improve time to value: provides out of box configs
- Enable plug & play dpeloyments for specific use cases
- Allows medium/large deployments sharing configs & content across multiple worker groups
- https://packs.cribl.io
- Allows ability to quickly allow troubleshooting by ease of replicating setup

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

### Filters
- Used in:
- - Routes: to select a stream of the data flow
- - Functions: to scope or narrow down the applicabilty of the function
- Filters are expressions that must evaluate to either true (or truthy) or false (or falsy)
|Truthy|Falsy|
|---|---|
|true|false|
|42|null|
|-42|undefined|
|3.14|0|
|"foo"|Nan|
|Infinity|''|
|-Infinity|""|
 
 
 
 
 
 
- Example:
- - ```source.endsWith('.log') || type=='vpcflow'```
