# Message Worker Plugin - User Guide

Welcome to the Message Worker Plugin (called JMS Worker before version 1.5.0)! This guide will help you get the most out of managing your JMS message queues and Apache Kafka topics directly from IntelliJ IDEA.

## Table of Contents
1. [Installation & Setup](#installation--setup)
2. [Quick Start](#quick-start)
3. [Managing Connections](#managing-connections)
4. [Browsing Queues](#browsing-queues)
5. [Sending Messages](#sending-messages)
6. [Message Templates](#message-templates)
7. [Topics & Subscriptions](#topics--subscriptions)
8. [Apache Kafka](#apache-kafka)
9. [Message History](#message-history)
10. [Advanced Operations](#advanced-operations)
11. [Tips & Tricks](#tips--tricks)
12. [Troubleshooting](#troubleshooting)

---

## Installation & Setup

### Installing the Plugin

1. Open IntelliJ IDEA
2. Go to **Preferences/Settings** → **Plugins**
3. Search for "Message Worker"
4. Click **Install** and restart IDE

Or install manually:
1. Download the plugin JAR file
2. Go to **Preferences/Settings** → **Plugins** → **⚙️** → **Install Plugin from Disk**
3. Select the JAR file
4. Restart IDE

### Accessing the Plugin

Once installed, you'll see a **Message Worker** tool window at the bottom of your IDE. Click on it to activate.

---

## Quick Start

### Step 1: Add a Connection

1. Click the **+** button in the Connections toolbar
2. Fill in connection details:
- **Name**: Give your connection a meaningful name (e.g., "Dev Artemis")
- **Type**: Select "Artemis", "ActiveMQ 5", or "IBM MQ"
- **Host**: Broker hostname
- **Port**: Broker port (usually 61616 for Artemis/ActiveMQ 5, 1414 for IBM MQ)
- **Username/Password**: Leave empty if not required
3. Click **Test Connection** to verify
4. Click **OK** to save

### Step 2: Browse a Queue

1. In the Message Worker tool window, you'll see your connections in a tree
2. **Click the arrow** to expand a connection (queues load on expand)
3. Click on a queue to browse its messages
4. Messages appear in the table on the right
5. Click a message to see details in the split panel below

### Step 3: Send a Message

1. Right-click on a queue → **Send Message**
2. Or click the **Send** button in toolbar
3. **Select connection and queue** from dropdowns (only connected servers shown)
4. Fill in the message body
5. Click **Send**

---

## Managing Connections

### Connection Types

#### Apache ActiveMQ Artemis
- **Host/Port**: Direct TCP connection
- **Credentials**: Optional username/password
- **Jolokia** (optional): Enable for queue/topic creation/deletion and topic discovery

#### ActiveMQ Classic 5.x
- **Host/Port**: OpenWire connection (usually 61616)
- **Credentials**: Optional for JMS, often required for web/Jolokia management
- **Jolokia**: Use for queue/topic discovery and broker admin actions
- **Default Jolokia path**: `/api/jolokia`

#### IBM MQ
- **Host/Port**: Queue manager connection
- **Queue Manager**: Queue manager name
- **Channel**: Channel name (usually "DEV.APP.SVRCONN")
- **CCSID**: Character encoding (usually 819 for UTF-8)
- **Authentication**: Choose from four modes:
  - **None** - no credentials
  - **Basic** - username + password
  - **mTLS** - certificate-based only (keystore/truststore, no password)
  - **Basic + mTLS** - both username/password and client certificate

#### IBM MQ - mTLS (mutual TLS) setup

When **mTLS** or **Basic + mTLS** is selected, an **SSL / TLS** section appears:

| Field | Description |
|-------|-------------|
| **Keystore** | Path to `.p12` or `.jks` file containing the client certificate and private key |
| **Keystore type** | `PKCS12` or `JKS` |
| **Keystore password** | Password for the keystore file |
| **Truststore** | Path to truststore file (optional - leave empty to use the JVM default trust store) |
| **Truststore type** | `PKCS12` or `JKS` |
| **Truststore password** | Password for the truststore file |
| **Cipher suite** | TLS cipher suite name (see below) |
| **Use IBM cipher mappings** | Controls cipher name format (see below) |

**Cipher suite and IBM cipher mappings:**

- If your broker uses **JSSE cipher names** (e.g. `TLS_AES_256_GCM_SHA384`): **uncheck** "Use IBM cipher mappings"
- If your broker uses **IBM cipher names** (e.g. `*TLS13ORHIGHER`, `SSL_RSA_WITH_AES_256_CBC_SHA256`): **check** "Use IBM cipher mappings"

Typical Spring Boot setup (`use-i-b-m-cipher-mappings: false`, cipher `*TLS13ORHIGHER`):
1. Set **Authentication** to **mTLS**
2. Set **Keystore** to your `.p12` file path and enter the keystore password
3. Set **Cipher suite** to `*TLS13ORHIGHER`
4. **Uncheck** "Use IBM cipher mappings"
5. Leave Truststore empty (unless your broker uses a self-signed or private CA certificate)

#### Apache Kafka
- **Bootstrap servers**: `host:port`, comma-separated for several brokers (e.g. `broker1:9092,broker2:9092`). Replaces Host/Port.
- **Security**: protocol `PLAINTEXT`, `SSL`, `SASL PLAINTEXT` or `SASL SSL`.
  - **SASL**: mechanism `PLAIN`, `SCRAM-SHA-256` or `SCRAM-SHA-512`, plus username and password.
  - **SSL / SASL SSL**: the SSL / TLS section appears. Set the truststore when the broker certificate is not trusted by the JVM default trust store. Add a client keystore only for mutual TLS.
- **Kafka Settings**: *Show internal topics* lists `__consumer_offsets`, `_schemas`, `_confluent-*` and similar topics, which are hidden by default.
- **New record format** (in Kafka Settings): the key and value format the send dialog starts with. *Like the records in the topic* (default) encodes a new record the way the topic's records already are, see [Sending Records](#sending-records). *Schema Registry when the `<topic>-key` / `<topic>-value` subject exists* switches each part whose subject is registered. *Always text* leaves both on Text.
- **Schema Registry URL** (in Kafka Settings, optional): the address of a Confluent Schema Registry, e.g. `http://localhost:8081`. With it, Avro and JSON Schema records are shown and sent as JSON. Username and password are only needed for a registry with basic authentication. An `https` registry uses the truststore and keystore of the SSL / TLS section. **Test Connection** checks the registry too.
- See [Apache Kafka](#apache-kafka) for how Kafka topics behave in the plugin.

### Connect and Disconnect

- **Expanding** a connection node (click the arrow) connects if not yet connected and loads the queue list.
- **Double-clicking** a disconnected connection node reconnects and loads queues immediately.
- **Right-click → Disconnect** explicitly disconnects. Once disconnected, the plugin will not reconnect in the background - the "Disconnected" state is sticky until you explicitly reconnect.
- The connection node collapses on disconnect and shows a "Loading..." placeholder so the expand arrow stays visible for the next connect.

### Lazy Loading

Connections now use **lazy loading**:
- All connections appear immediately in the tree
- Queues and topics load only when you **expand** a connection
- Each connection loads independently (no waiting for slow servers)
- Loading indicator shows "Loading..." while fetching queues

### Testing Connections

Before saving, always test your connection:
1. Fill in the connection details
2. Click **Test Connection**
3. You'll see "Connection successful" or an error message
4. Only save if the test passes

### Duplicating a Connection

In **Settings → Tools → Message Worker**, select an existing connection in the list and click the **Copy** icon in the toolbar above the list. A new-connection dialog opens with every field - including credentials, keystore paths, and SSL settings - pre-filled from the source. The name is set to `<original> (copy)` (or `(copy 2)`, `(copy 3)`, … if a copy already exists) so you can save immediately. Tweak whatever differs (typically host or environment-specific values) and hit OK. Useful for cloning a dev connection into a UAT one without retyping everything.

### Context Menu Actions

Right-click on a **connection** for:
- **Refresh Queues** - Reload queue list
- **Edit Connection** - Modify settings
- **Create Queue** - Create new queue (Artemis or ActiveMQ 5 + Jolokia)
- **Create Topic** - Create new topic (Artemis or ActiveMQ 5 + Jolokia)
- **Send Message to...** - Open send dialog
- **Connect/Disconnect** - Toggle connection state
- **Delete Connection** - Remove from settings

Right-click on a **queue** for:
- **Browse Queue** - Load messages
- **Send Message** - Send to this queue
- **Delete Queue** - Remove queue (Artemis or ActiveMQ 5 + Jolokia)
- **Purge Queue** - Delete all messages

Right-click on a **topic** for:
- **Subscribe to Topic** - Opens subscribe dialog with pre-filled topic name
- **Publish to Topic** - Opens send dialog in topic mode
- **Delete Topic** - Remove topic (Artemis or ActiveMQ 5 + Jolokia)

---

## Browsing Queues

### Tree Navigation

The left panel shows a tree structure with Queues and Topics as separate categories:
```
📁 Connections
  ├── 🟢 Dev Artemis (7)
  │   ├── 📋 Queues (5)
  │   │   ├── orders.queue (42)
  │   │   ├── payments.queue (0)
  │   │   └── ⚠️ DLQ (3)
  │   └── 📡 Topics (2)
  │       ├── news.topic
  │       └── events.topic ●
  └── 🔴 Prod IBM MQ (not connected)
```

- 🟢 Green = Connected
- 🔴 Red = Disconnected
- ⚠️ Warning = DLQ with messages
- ● Green dot = Topic has active subscription
- Numbers in parentheses = message count (auto-refreshed every 2 seconds)

### Tree Stays Expanded

Queue count refresh now updates individual queue labels without collapsing the tree. Your expanded connections, queues, and topics categories stay open.

### Simplified Queue Names

Queue names are simplified for readability:
- `orders::orders` displays as just `orders`
- `address::queue` displays as `queue` (when address equals queue name)
- Full name is used for API calls

### Message Count vs Loaded

The header shows actual vs loaded count:
- **"Queue: orders (580 messages, showing 100)"**
- Click **"Load more (480 remaining)..."** to load additional messages

### Filtering Messages

Use the filter field above the message table:
- Type to filter by message ID, correlation ID, body content
- Results update in real-time
- Clear to see all messages

### Speed Search in Tree

Just start typing while the tree is focused:
- Matches queue names and topic names
- Press Enter to select match
- Press Escape to cancel

### Message Details

Click any message to see the **split detail panel**:

**Left side - Body:**
- Syntax highlighted (JSON/XML auto-detected)
- Icon buttons above the body: Copy (copies the **formatted** content), Plain text, JSON, XML
- **Decoded**, **Raw** and **Bytes** switch the view, the active one is shown pressed
- The arrow keys keep moving through the message list while the detail follows the selection, and refreshing the list (also by auto refresh) keeps the selected messages

A message you send blinks green in the list of its queue or topic, so you see what was added.

**Right side - Headers & Properties:**
- Message ID, Correlation ID, Timestamp
- Type, Priority, Delivery Mode
- Body type (TEXT/BYTES) and size
- Custom properties

### BYTES Messages

BYTES messages are **automatically decoded**:
- Base64 content converted to readable text
- Original encoding respected (UTF-8, CP912, etc.)
- Headers show both Base64 and decoded size

---

## Sending Messages

### Connection & Queue Selection

The send dialog includes **dropdowns**:
- **Connection**: Only shows connected servers
- **Destination**: Switch between Queue and Topic mode
- **Queue/Topic**: Dynamically loads items for selected connection
- You can also type a name manually

### Basic Message

1. Select connection and destination (queue or topic)
2. Choose **Message Format**: TextMessage or BytesMessage
3. Type your message body
4. Click **Send**

### Message Headers

Expand **Message Headers** section:
- **Correlation ID**: For request/reply patterns
  - ☑️ **Convert ASCII to HEX**: For IBM MQ on AS/400
- **Type**: JMS message type identifier
- **Priority**: 0-4 (normal), 5-9 (expedited)
- **Persistent**: Survives broker restart

### ASCII to HEX Conversion

For IBM MQ on AS/400 that requires HEX correlation IDs:
1. Enter correlation ID as ASCII (e.g., `ORDER123`)
2. Check **"Convert ASCII to HEX"**
3. Plugin converts to HEX before sending (e.g., `4F524445523132333`)

### Extended Options

Expand **Extended Options** section:
- **Reply-To Queue**: Where responses should go
- **Time-To-Live (ms)**: Message expiration (0 = never)
- **Group ID**: For message grouping (JMSXGroupID)

### Custom Properties

Expand **Custom Properties** section:
- One property per line: `key=value`
- Example:
  ```
  environment=production
  version=2.1
  requestId=abc123
  ```

### Message Format & Encoding

- **TextMessage**: Plain text (default)
- **BytesMessage**: Binary with encoding selection
  - UTF-8 (default)
  - UTF-16, ISO-8859-1, ASCII
  - CP1252, CP912, EBCDIC-US (for mainframe)

### Saving to History

Expand **History** section:
- ☑️ **Save to history**: Store for later reuse
- **Label**: Optional name for easy finding

---

## Message Templates

The **Templates** tab lets you save, organize, and reuse frequently sent messages.

### Creating a Template

1. Open the **Templates** tab at the bottom of the tool window
2. Click **+** (New Template) in the toolbar
3. Fill in the template:
   - **Name**: Descriptive name shown in the list
   - **Connection** and **Destination**: Optional - a template without these is portable across environments
   - **Message body**: Can contain variables (see below)
   - Message type, priority, TTL, correlation ID, and custom properties are all saved
4. Click **Save**

### Using a Template

1. In the Templates tab, double-click a template (or right-click → **Use**)
2. If the template has a connection and destination bound, the Send Message dialog opens pre-filled and ready to send
3. If the template is missing a connection or destination, a small target dialog appears asking only for the missing values - fill them in and continue

### Variable Substitution

Template bodies support built-in and custom variables:

| Variable | Value |
|----------|-------|
| `{{uuid}}` | Random UUID generated at send time |
| `{{timestamp}}` | Current ISO-8601 timestamp |
| `{{myVar}}` | Custom variable - prompted at send time |

Example body:
```json
{"orderId": "{{uuid}}", "createdAt": "{{timestamp}}", "customerId": "{{customerId}}"}
```
Custom variables (`{{customerId}}` in the example) are collected in the Variables dialog before sending.

### Managing Templates

Right-click a template for:
- **Use** - open in Send Message dialog
- **Edit** - modify the template
- **Duplicate** - copy for quick variation
- **Delete** - remove
- **Mark as Favorite** - pin to the top of the list (shown with star icon)

---

## Topics & Subscriptions

### Overview

The **Topics** tab provides real-time topic subscription management for Artemis connections. Topics are auto-discovered via Jolokia (MULTICAST addresses) and displayed under each connection in the tree. ActiveMQ Classic 5.x also supports topic discovery, publishing, and topic create/delete via Jolokia.

### Subscribing to a Topic

**From the Topics tab:**
1. Click the **▶ Subscribe** button in the toolbar
2. Select connection and topic from dropdowns
3. Optionally enable **Durable subscription** and enter a subscription name
4. Optionally enter a **message selector** (JMS SQL-like syntax)
5. Click **Subscribe**

**From the connection tree:**
1. Expand a connection → Topics category
2. Right-click on a topic → **Subscribe to Topic**
3. The subscribe dialog opens with the topic pre-filled
4. After subscribing, the plugin switches to the Topics tab automatically

### Subscription Indicators

Active subscriptions are visible in two places:
- **Connection tree**: Topics with active subscriptions show a green **●** indicator
- **Topics tab**: Subscription list shows status icons (● active, ○ stopped, ✖ error)

These indicators synchronize automatically - stopping or removing a subscription from the Topics tab immediately updates the tree, and vice versa.

### Viewing Live Messages

1. In the Topics tab, click on a subscription in the left list
2. Incoming messages appear in the table in real-time
3. Click a message to see full details (body + headers)
4. Use **Auto-scroll** toggle to follow latest messages

### Unread Message Badge

When the Topics tab is not active (you're on Browser or History), incoming messages are counted as unread:
- The tab title changes from **Topics** to **Topics (5)** showing the unread count
- The badge resets automatically when you switch to the Topics tab
- The Message Worker tool window also activates to draw your attention on the first unread message

### Publishing to a Topic

**From the connection tree:**
1. Right-click on a topic → **Publish to Topic**
2. The Send Message dialog opens in **Topic mode** with the topic pre-selected
3. Enter your message and click **Send**

**From the Send Message dialog:**
1. Change the **Destination** dropdown from "Queue" to "Topic"
2. The destination list switches to available topics
3. Select or type a topic name and send

### Managing Subscriptions

Right-click on a subscription in the Topics tab for:
- **Stop Listening** - Pause the subscription (buffered messages kept)
- **Clear Messages** - Remove buffered messages
- **Remove Subscription** - Stop and remove entirely
- **Remove Durable Subscription from Broker** - Delete the durable subscription on the broker (pending messages lost)

### Topic Management

- **Artemis + Jolokia**:
  - **Create Topic**: Right-click on connection → Create Topic, or right-click on Topics category → Create Topic
  - **Delete Topic**: Right-click on topic → Delete Topic (also removes durable subscriptions)
- **ActiveMQ 5 + Jolokia**:
  - **Create Topic**: Right-click on connection → Create Topic
  - **Delete Topic**: Right-click on topic → Delete Topic

---

## Apache Kafka

Kafka topics work like queues in the plugin: they sit in the connection tree, open in the same browser, and use the same send dialog, templates and history. The differences below come from how Kafka works.

### Topics in the Tree

A Kafka connection shows **Topics** with every topic and the number of records it currently holds (all partitions together). Kafka keeps records after they are consumed until retention removes them, so this number is the content of the topic, not a backlog, and large numbers are not highlighted. Right-click a topic for **Browse Topic**, **Send Message**, **Live Tail...**, **Topic Details...**, **Copy Records to...**, **Create Topic...**, **Delete Topic** and **Purge Topic**.

Below Topics there are two more categories. They stay collapsed and are loaded only when you open them, so a connection you only browse is not slowed down:

- **Consumer Groups**: every consumer group of the cluster with its state and total lag, for example `billing (stable lag 42)`. Double-click a group to open it (see [Consumer Groups](#consumer-groups)).
- **Schema Registry**: the subjects of the connection's Schema Registry, shown only when the connection has a registry URL. Double-click a subject to open it (see [Schema Registry Subjects](#schema-registry-subjects)).

Right-click either category for **Refresh**. The tree search field filters groups and subjects as well.

### Browsing a Topic

When a Kafka topic is selected, a second row above the table sets where reading starts:

| Read | What you get |
|------|--------------|
| **Newest** | The latest records (default) |
| **Oldest** | The earliest records still in the topic |
| **From offset** | Records from the given offset on, in every selected partition. Press Enter after typing the offset. |
| **From time** | Records written at or after the given time. Accepts `yyyy-MM-dd HH:mm`, seconds and milliseconds optional, a date alone, or epoch milliseconds. Press Enter to apply. |

**Partition** limits reading to one partition. Leave it empty for all partitions. **Load more** and **Auto** refresh work as for queues.

The table columns change to **Offset**, **Key**, **Timestamp**, **Format**, **Partition** and **Preview**. The detail panel shows the record (topic, partition, offset, key, timestamp, value size in bytes) and its headers. Values that are not UTF-8 text are shown as BYTES, use the **Bytes** view for the hex dump. A record with a null value (tombstone) is marked as such.

**Decoded**, **Raw** and **Bytes** choose how you look at a value. **Decoded** shows the text, or the JSON of a Schema Registry record. **Raw** (Base64) and **Bytes** (hex dump) always show the value exactly as it is stored in the topic, also for records shown as JSON.

Browsing never consumes: the plugin reads without a consumer group and commits nothing, so the applications on the topic are not affected. The filter field also searches in record keys.

### Searching a Topic

The filter field only looks at the records already loaded. To search the whole topic on the broker, like AKHQ does, click **Search** at the end of the read-position row. A search row opens:

| Field | Matches |
|-------|---------|
| **Key** | the record key |
| **Value** | the value as shown in Decoded (the JSON of a registry record). Binary values are searched as text fragments and as `0x` hex. |
| **Header** | a header name (contains) and a header value |
| **Until** | leaves out records newer than this time |
| **Scan/partition** | how many records are read from each partition, starting at the read position (default 10000) |

Each term can be **contains** or **not contains** (both ignore case) or **equals** (exact). Every filled term must match. Press Enter or **Search**. The header then shows how many records matched and how many were read, for example `3 matches in 12000 records scanned`. The read position and partition above still decide where the search starts: **Newest** searches the newest records, **Oldest** or **From time** search forward. A search stops after one minute and says so, narrow the read position or the scan then. **Clear** empties the terms, closing the row with **Search** returns to the normal browse.

### Record Actions

Right-click records in the browser:

- **Export Selected...** and **Export All Loaded...** save records to a JSON file: partition, offset, timestamp, key, value (a JSON value stays JSON), headers, the registry subjects, and the value bytes as stored (Base64).
- **Delete by Key (Tombstone)...** writes a record with the same key and an empty value into the same partition. On a compacted topic (`cleanup.policy=compact`) Kafka then removes every earlier record with that key the next time the log is compacted, not immediately. On other topics the plugin refuses it, because there a tombstone would remove nothing.

### Schema Registry (Avro and JSON Schema)

Records written by applications that use the Confluent Schema Registry carry a schema id in front of the data. With a **Schema Registry URL** in the connection, the plugin loads that schema and shows the record as JSON:

- The **Format** column says **AVRO** or **JSON**, and the detail panel shows the schema id, the record name and the subject.
- Keys registered in the Schema Registry are shown as JSON too.
- Timestamps and dates are shown as ISO-8601 text (`2026-09-30T10:15:30.123456Z`, `2026-09-30`), decimals as numbers, bytes as Base64, and optional fields without the Avro type wrapper, the way AKHQ shows them.
- Without a registry URL such records stay BYTES, and the **Decoded** view starts with a note that they are registry records, which schema id they use and where to set the registry URL.
- Protobuf records are recognised and shown as bytes.

### Sending Records

For a Kafka connection the send dialog shows:

- **Key**: the record key. Empty means no key. Records with the same key always go to the same partition.
- **Key format** and **Value format**: how the key and the body become the bytes of the record. For a new message the dialog starts with the formats the topic's records already have (the majority of its newest 20 records), so a record sent with the defaults looks like the ones the topic's producers write: Schema Registry with the subject those records use, text or hex. The key format changes only when the topic's records carry keys. A topic whose records have plain string keys keeps a text key, even when a `<topic>-key` subject is registered. An empty topic starts with Schema Registry when its `<topic>-value` subject exists. A format you pick yourself is kept, and resends and templates keep their own formats. The connection setting **New record format** changes this behaviour.
  - **Text**: the text in the chosen **Encoding**, like a JMS text or bytes message.
  - **Bytes (hex)**: the text is hex digits (spaces and line breaks are ignored), sent byte for byte. Use it for binary values.
  - **Schema Registry**: the text is JSON, sent as Avro or JSON Schema with the latest schema of the chosen **subject**. The subject list comes from the registry, and `<topic>-value` / `<topic>-key` is filled in by default. JSON that does not fit the schema is refused before anything is sent, and the message names the field, e.g. `eventMetadata.eventId: missing required field`. Sending never registers a schema, that is done only in the Schema Registry view.
- **Partition**: optional target partition. Empty lets Kafka choose (from the key, or spread when there is no key).
- **Custom Properties / Headers**: one `key=value` per line, sent as Kafka record headers (UTF-8).
- **Kafka: Timestamp and Batch** (collapsed):
  - **Timestamp**: the record timestamp (`yyyy-MM-dd HH:mm[:ss]`, a date, or epoch milliseconds). Empty means now.
  - **Batch**: every non-blank line of the body is sent as its own record, with the same format, headers and partition. With a **Key separator**, the text before its first occurrence on a line is that record's key, e.g. with `:` the line `id-7:{"a":1}` has key `id-7`. Lines without the separator use the **Key** field. Every line is checked first, so a line that does not fit (bad hex, JSON that does not match the schema) sends nothing and the error names the line. A batch holds up to 10000 records and is kept in the send history as one entry.

The JMS-only fields (destination type, correlation ID, priority, persistence, reply-to, TTL, group ID) are hidden. **Resend** and **Send Similar Message** keep the key and headers of the record and pick the format that sends it unchanged: a registry record opens as JSON with its subject, a binary record opens as hex of its exact bytes. Templates keep the key (with variables), the partition, both formats and the subjects. The send history shows them in the detail and the history search also matches keys.

### Live Tail

**Live Tail...** (or **Subscribe** in the Topics tab) follows new records:

- Without **Consumer group** the plugin reads every partition from its current end. Nothing is stored on the broker.
- With **Consumer group** the plugin reads as the given group id and commits offsets, so the next tail with the same group id continues where the previous one stopped. **Remove Subscription** only stops it in the plugin and keeps the group. Right-click it and choose **Delete Consumer Group from Broker** to delete the group and its committed offsets (records in the topic stay).

JMS message selectors have no Kafka equivalent, so the selector field is disabled.

### Topic Administration and Copying

- **Create Topic...** asks for the name, the number of partitions and the replication factor (empty = broker default). The collapsed **Advanced** group sets the cleanup policy (delete, compact, both), the retention and any other topic configuration as `name=value` lines.
- **Purge Topic** deletes every record currently in the topic. This affects every consumer, not only the plugin, and the confirmation says so.
- **Delete Topic** removes the topic with all its records.
- **Copy to Topic...** (right-click selected records in the browser) re-produces the records into another topic with key, value and headers unchanged. The target partition follows the key.
- **Copy Records to...** (right-click a topic in the tree) copies many records at once into a topic on the same or another Kafka connection: from the oldest, the newest, an offset or a time, one or all partitions, up to a number of records per partition. Keys, values and headers are copied byte for byte. **Keep partition numbers** puts every record into the partition with the same number (the target needs at least as many partitions), **Keep record timestamps** keeps the source times. Records written with a Schema Registry keep their schema id, so on another cluster the target registry must know the same ids.
- Moving or deleting single records is not possible in Kafka, so **Move** and **Delete** are disabled for Kafka topics. Use **Purge Topic** to empty a topic, or **Delete by Key (Tombstone)** on a compacted topic.

### Topic Details

**Topic Details...** on a topic opens its page:

- **Partitions**: leader, replicas, in-sync replicas, first and next offset, records and size on disk per partition, and a summary with the replication factor, the total size and the number of under-replicated partitions.
- **Configuration**: the values set on the topic. **Show broker defaults** also lists what the topic inherits. **Edit...** (or a double-click) changes a value, **Add...** sets a new one as `name=value`, **Reset to Default** removes the selected topic values so the broker defaults apply again. Every change is confirmed and applies to the topic immediately. Hover a name for the broker's description.
- **Consumer Groups**: the groups that committed offsets on the topic, with their lag on this topic. Double-click one to open it.
- **Increase Partitions...** raises the partition count. Kafka cannot reduce it later, and records with a key may then land in another partition than before.

### Consumer Groups

Double-click a group in **Consumer Groups** (or in the Consumer Groups tab of Topic Details). The dialog shows the state, the coordinator, the partition assignor and the total lag, and two tabs:

- **Offsets**: per topic and partition the committed offset, the end offset, the lag, and the client and host that own the partition.
- **Members**: member id, client id, host, static instance id and assigned partitions.

Actions:

- **Reset Offsets...** moves the committed offsets of the selected partitions, of all partitions of one topic, or of the whole group to the **earliest** retained record, the **latest** (end), the first record **at a time**, a **specific offset**, or **shifted by** a number of records (negative = read again, positive = skip). Offsets are kept inside the retained range. Kafka only moves the offsets of a group without active members, so stop its consumers first, the plugin says so otherwise.
- **Delete Topic Offsets...** removes the group's committed offsets of one topic, the group itself stays.
- **Delete Group...** deletes the group with all its committed offsets. The records in the topics stay.

### Schema Registry Subjects

Double-click a subject in **Schema Registry**:

- **Version** picks a version, the id, type and record name are shown next to it and the schema below.
- **Compatibility** shows the subject's level or `(global default)`. Choose a level and **Apply** to set it on the subject, or choose `(global default)` and **Apply** to use the registry's default again.
- **New Version...** opens the schema editor with the current version. **Check Compatibility** asks the registry whether the new schema is compatible with the latest version and shows its reasons when it is not. **Register** checks that the schema parses and registers it.
- **Delete Version...** and **Delete Subject...** remove a version or the whole subject. Both are soft deletes unless **Delete permanently** is checked. Producers that use a deleted subject fail to send until it is registered again.

Right-click the **Schema Registry** category for **New Subject...** (registers the first version of a new subject, `<topic>-value` of the selected topic is suggested), **Find Schema by ID...** (the schema behind the id at the start of a registry record, and which subjects and versions use it) and **Global Compatibility...**.

---

## Message History

### Accessing History

Click the **History** tab at the bottom of the tool window.

### Same Detail View

History has the **same split panel** as the queue browser:
- Body on the left with formatting
- Headers on the right
- JSON/XML format buttons
- Copy formatted content

### BYTES in History

BYTES messages are properly decoded:
- **Format** column shows TEXT or BYTES
- Detail view shows decoded content
- Edit & Send uses decoded body

### Filtering History

- **Connection**: Filter by source connection
- **Queue**: Filter by target queue
- **Search**: Find by body, label, or correlation ID

### Resending Messages

1. Select a message
2. Click **Resend** → Confirms and sends immediately
3. Or click **Edit & Send** → Opens in send dialog for modifications

### Managing History

- **Set Label**: Add/edit label for organization
- **Delete**: Remove single message
- **Clear Old**: Remove messages older than N days
- **Clear All**: Delete entire history

---

## Advanced Operations

### Creating Queues

Requires Jolokia enabled in connection settings:

1. Right-click on connection → **Create Queue...**
2. Enter queue name (e.g., `my.new.queue`)
3. Click OK
4. Queue appears in tree immediately (targeted reload, no tree collapse)

### Creating Topics

Requires Jolokia enabled:

1. Right-click on connection → **Create Topic...**
2. Enter topic name (e.g., `my.new.topic`)
3. Click OK
4. Topic appears under the Topics category immediately

### Deleting Queues/Topics

1. Right-click on queue/topic → **Delete Queue** / **Delete Topic**
2. Confirm deletion
3. ⚠️ Messages in queue/topic are lost!

### Moving Messages

1. Select one or more messages (Ctrl+click for multi-select)
2. Right-click → **Move to Queue...**
3. Select target queue
4. Messages removed from source, added to target

### Copying Messages

1. Select messages
2. Right-click → **Copy to Queue...**
3. Select target queue
4. Original stays, copy created in target

### Deleting Messages

1. Select one or more messages (Ctrl+click for multi-select)
2. Right-click → **Delete Messages**, or press the **Delete** key
3. Confirm deletion
4. The selected message(s) are permanently removed and the queue view refreshes. The status bar reports how many were deleted, not found, or failed
5. ⚠️ Cannot be undone!

Supported on IBM MQ and Apache Artemis. (On ActiveMQ Classic, single-message delete is not available - use Purge Queue instead.)

### Purging Queues

1. Right-click on queue → **Purge Queue**
2. Confirm deletion of ALL messages
3. ⚠️ Cannot be undone!

### Drag & Drop

Drag messages from table to queue in tree:
- Moves messages to target queue
- Visual feedback during drag

---

## Tips & Tricks

### Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| F5 | Refresh current view |
| Ctrl+F | Focus filter/search field |
| Delete | Delete selected (in history) |
| Enter | View message details |
| Ctrl+C | Copy selected message body |
| Ctrl+A | Select all messages |

### Splitter Customization

Drag the splitters to resize panels:
- Hover over splitter → shows accent color
- Drag to resize
- Double-click to reset to default

### Organizing with Labels

Use descriptive labels:
- `BUG-123: order processing error`
- `TEST: batch import scenario 5`
- `TEMPLATE: standard order request`

### Message Templates

Use the **Templates** tab for messages you send repeatedly:
- Save with variable placeholders (`{{uuid}}`, `{{timestamp}}`, custom vars) to get fresh values on every send
- Leave connection/destination unbound for payloads you use across environments - the target dialog asks for them at send time
- Mark frequently used templates as Favorites so they stay at the top of the list

### Performance Tips

For large queues (1000+ messages):
- Use filters to narrow results
- Don't load more than needed
- Close unused connections
- Periodically clean old history

### Reading the Queue Tree

Each queue in the connection tree shows its current depth as `(N)` next to the name. The icon, name color, and count color jointly reflect the queue's state. **Hover anywhere on the row** (over the icon or the text) to see a tooltip explaining the meaning and what action, if any, it suggests.

| State | Icon | Name + count | When |
|---|---|---|---|
| Empty | default queue | grey `(0)` | Queue has no messages. |
| Normal | default queue | regular `(N)` | 1-100 messages - normal operation. |
| Backlog | information | bold orange `(N)` | More than 100 messages - possible backlog, check that consumers are running. |
| Dead-letter / error | warning | bold red `(N)` | Queue name contains `DLQ`, `dead`, or `error` (case-insensitive) AND it has messages. These were rejected by their original destination and need investigation. |
| Depth unknown | error | italic grey **name** + `(?)` | The plugin couldn't read this queue's depth - the broker rejected the depth request (typically `MQRC_NOT_AUTHORIZED` 2035 on IBM MQ). Browse / send / subscribe still work if you have those rights. To restore the depth badge, ask your MQ admin to grant `+dsp` (and `+inq`) on this queue. |

### Column widths persist across restarts

Resize a column in any data table - queue browser, message history, templates, live topic messages - and the new width is saved automatically. The next time you open the IDE the table picks up where you left off. Each table remembers its own widths independently.

### Performance & Polling Settings (v1.4.0+)

A new section in **Settings → Tools → Message Worker → Performance** controls the background polling and timeout behavior. Defaults are tuned for daily use. Tweak only if you see broker-side log noise or want a snappier UI.

| Setting | Default | What it does |
|---|---|---|
| Auto-refresh queue counts in tree | on | Master switch for the background queue depth poller. Turning it off stops all "(N)" badge updates in the tree but does not affect Browse / Send / Subscribe. |
| Refresh interval | 5 s | How often the poller runs. The poller hits only **expanded** connection nodes - collapse a connection to silence its polling without disabling it globally. |
| Topic message coalescing window | 150 ms | Live topic messages are batched per window before being pushed to the UI. Higher values keep the UI smoother under heavy throughput at the cost of slightly delayed visibility. |
| Connection close timeout | 3 s | Per-connection wall-clock cap when shutting the IDE down or clicking Disconnect / Disconnect All. Prevents a hung broker from stalling the UI. |
| Jolokia HTTP connect timeout | 5 s | Applies to Apache Artemis and ActiveMQ Classic management calls. |
| Jolokia HTTP read timeout | 5 s | Applies to Apache Artemis and ActiveMQ Classic management calls. |

**IBM MQ note:** if your channel rejects queue depth probes with `MQRC_NOT_AUTHORIZED` (2035), the plugin pauses further probes - no more per-tick log spam in the broker. The pause is per-queue when only individual queues are inaccessible (other queues on the same connection keep updating). It widens to the whole connection only when the channel itself has no admin access. Either form clears the next time you reconnect. If you want depth polling permanently silent for a specific environment, turn the global "Auto-refresh queue counts" off in Settings.

**IBM MQ depths showing as zero or missing?** The batched wildcard depth call needs **`+dsp`** (display) authority on each queue, in addition to the usual `+inq`. Without `+dsp`, the broker silently drops queues from the response. Ask your MQ admin to grant:

```
setmqaut -m <QM> -t queue -n 'YOUR.PREFIX.**' -p <user> +dsp
```

The plugin falls back to a per-queue lookup for queues missing from the batched response, which works with just `+inq` but is one round-trip per queue - slower for big queue lists.

### How Credentials Are Stored

The plugin stores all sensitive values exclusively in the IDE's credential store (`PasswordSafe`), the same mechanism IntelliJ uses for Git and database credentials. This applies to:

- Connection passwords (Basic auth)
- Keystore passwords (mTLS)
- Truststore passwords (mTLS)

The plugin's settings file holds only non-sensitive metadata: connection name, host, port, channel, queue manager, paths to keystore/truststore files, cipher suite. The settings file never contains your passwords.

**One-time migration:** if you used an older version that may have left plaintext credentials in the settings file, the first launch of 1.4.0 moves any plaintext value into the credential store and rewrites the settings file without them. No action needed on your side.

**Recommendation for sensitive environments:** the default credential-store backend on Linux is a file inside your IntelliJ config directory. For better protection, switch to the OS keyring under **Settings → Appearance & Behavior → System Settings → Passwords → "In native keychain"**:

- Linux: libsecret (GNOME Keyring / KWallet)
- macOS: Keychain
- Windows: Credential Manager

Note that the keystore file itself (`.p12` / `.jks`) lives wherever you placed it on disk. The keystore password protects the private key inside, but file-level access controls are still your responsibility. Keep the keystore out of source control and out of cloud sync directories.

### Working with Multiple Environments

Create connections for each environment:
- `DEV-Artemis` → development server
- `TEST-IBM-MQ` → testing queue manager
- `PROD-Artemis` → production (read-only recommended)

Color-code with labels to avoid mistakes!

---

## Troubleshooting

### Connection Issues

**"Cannot connect to broker"**
- ✔ Check hostname and port
- ✔ Verify network connectivity
- ✔ Check firewall rules
- ✔ Try Test Connection button

**"Authentication failed"**
- ✔ Verify username and password
- ✔ Check account is enabled on broker
- ✔ For IBM MQ: verify CCSID and channel

**Connection loads slowly**
- This is normal for remote/slow brokers
- Each connection loads independently
- Other connections remain usable

### Queue Issues

**"Cannot see queues"**
- ✔ Click arrow to expand connection
- ✔ Wait for "Loading..." to complete
- ✔ Check connection is still active
- ✔ Try Refresh Queues from context menu

**"Queue shows wrong count"**
- Counts auto-refresh every 2 seconds in the background
- Click on a queue to browse and get exact count
- Try Refresh Queues from context menu for immediate update

**"IBM MQ queues show 0 messages (mTLS / app-only channel)"**
- The plugin tries the fast PCF admin path first. If that channel does not have access to `SYSTEM.ADMIN.COMMAND.QUEUE`, it automatically falls back to a direct `MQOO_INQUIRE` open on each queue
- The fallback requires only BROWSE and INQUIRE rights on the queues, no PCF admin rights
- If counts still show 0, confirm the JMS user has at least BROWSE+INQUIRE on those queues

### Topic Issues

**"No topics visible"**
- ✔ Topics require broker management support with Jolokia enabled
- ✔ Check Jolokia URL in connection settings
- ✔ Artemis topics must use MULTICAST routing type on the broker

**"Publish to Topic does nothing"**
- ✔ Ensure the Send Message dialog shows "Topic" in the destination dropdown
- ✔ Verify the connection is active
- ✔ Right-click → Publish to Topic automatically selects topic mode

**"Subscription indicator not updating"**
- The tree syncs automatically with the Topics tab
- Try expanding/collapsing the connection to force refresh

### Message Issues

**"BYTES message shows Base64"**
- Should auto-decode in detail view
- Check encoding matches source
- Try different encoding in settings

**"Special characters corrupted"**
- Select correct encoding when sending
- CP912 for Cyrillic
- EBCDIC-US for mainframe

**"Copy gives wrong content"**
- Format first (JSON/XML button)
- Then use Copy button
- Copies formatted content

### Create Queue/Topic Fails

**"Jolokia required"**
- Enable Jolokia in connection settings
- Provide Jolokia URL (e.g., `http://localhost:8161/console/jolokia` for Artemis or `http://localhost:8161/api/jolokia` for ActiveMQ 5)
- Requires broker configuration

**"Invalid JSON request"**
- Update to latest plugin version
- Check Jolokia endpoint is accessible

### Database Issues

**"History not working"**
- Close all connections
- Delete: `~/.config/JetBrains/IntelliJIdea*/jms-worker/message-history.db`
- Restart plugin (recreates database)
- ⚠️ Deletes all history!

---

## What's New in v1.6.1

- **Kafka records sent like the topic's own**: the send dialog starts with the key and value format of the topic's records, keys included. The connection setting *New record format* switches to subject naming or to text. See [Sending Records](#sending-records).
- **Sent messages blink** in the list of their queue or topic, and a sent Kafka record shows its stored format right away.
- **Fixed**: the arrow keys move through the message list again, and refreshing the list keeps the selection.
- **Fixed**: the plugin unloads without a restart when it is disabled or updated (for updates from 1.6.1 on).
- **Changed**: the buttons above the message body look like the IDE's own.

## What's New in v1.6.0

- **Kafka administration like AKHQ**: consumer groups with lag and offset reset, search of a whole topic, topic details and configuration, Schema Registry subjects, batch send, record timestamps, tombstones, JSON export and copy between topics and clusters. Everything new sits in collapsed sections. See [Apache Kafka](#apache-kafka).

## What's New in v1.5.0

- **New name**: JMS Worker is now Message Worker. Your connections, passwords, history, templates and settings are kept.
- **Apache Kafka support**: Kafka connections next to Artemis, ActiveMQ Classic and IBM MQ. Browse topics from the newest or oldest record, from an offset or from a point in time, per partition, without consuming. Send records with key, headers and partition, follow topics live, create, purge, delete and copy. PLAINTEXT, SSL / mutual TLS, SASL PLAIN and SCRAM. See [Apache Kafka](#apache-kafka).
- **Schema Registry**: Avro and JSON Schema records are shown and sent as JSON. See [Schema Registry](#schema-registry-avro-and-json-schema).
- **Binary Kafka values in both directions**: send a key or value as hex, byte for byte, and see every value as stored in the Raw and Bytes views.
- **Fixed**: the preview of a binary message shows its size in bytes (it showed the length of its Base64 text).
- **Fixed**: closing the IDE no longer reports a memory leak in the plugin.
- **Fixed**: stopping a topic subscription now really stops receiving messages.
- **Fixed**: saving or editing a connection with a password no longer triggers an IDE error report.
- **Fixed**: Copy to Queue accepts a destination typed by hand.
- **Fixed**: a message deleted right after it was sent no longer reappears in the browser.

## What's New in v1.4.3

- **Maintenance release**: removes the internal IntelliJ Platform API flagged by JetBrains Marketplace verification (the plugin version is now injected at build time instead of read from a platform API). No functional changes - includes the v1.4.1 message-delete fix below.

## What's New in v1.4.2

- **Maintenance release**: internal compatibility cleanup for current IntelliJ Platform APIs (flagged by JetBrains Marketplace verification). No functional changes - includes the v1.4.1 message-delete fix below.

## What's New in v1.4.1

- **Fixed: deleting a message now works**. Selecting a message in the queue browser and choosing Delete (toolbar, right-click → Delete Messages, or the Delete key) previously did nothing - no error, the message stayed. Delete now permanently removes the selected message(s) on IBM MQ and Apache Artemis, refreshes the queue, and tells you in the status bar how many were deleted, not found, or failed. See "Deleting Messages" above. (Not available on ActiveMQ Classic - use Purge Queue there.)

## What's New in v1.4.0

- **Editor and tab-switch responsiveness**: the IDE no longer slows down while a connection is active in the background.
- **Batched queue depth refresh**: the tree's queue counts now refresh with a single broker call per connection per tick (previously one call per queue). Default refresh interval bumped from 2 s to 5 s.
- **Polls only what you can see**: collapsing a connection in the tree silences its background polling without disabling the global setting.
- **In-place tree updates**: refresh redraws only the queues whose count actually changed - the tree no longer flickers or collapses during refresh.
- **Topic message coalescing**: live topic messages are batched into a small window before reaching the UI. High-throughput topics no longer freeze the IDE.
- **Fast IDE shutdown and Disconnect**: closing connections is bounded by a hard timeout. A hung or unreachable broker can no longer stall IDE exit. Per-connection Disconnect over VPN that previously took 25-30 s is now sub-second.
- **IBM MQ - broker-side log flood fixed**: when a depth probe is rejected with `MQRC_NOT_AUTHORIZED` (2035), the plugin pauses probes - per-queue when only individual queues are inaccessible (other queues on the same connection keep updating), connection-wide only when the channel itself has no admin access. Cooldown clears on the next reconnect.
- **IBM MQ - mTLS connection setup unified**: removes a class of issues where background admin requests appeared to the broker as a different identity than the one configured by the user.
- **Security**: passwords (connection, keystore, truststore) are stored exclusively in the IDE credential store. A one-time migration scrubs any plaintext that older versions could leave in the settings file. See "How Credentials Are Stored" above.
- **New Performance settings**: Tools → Message Worker → Performance lets you tune refresh interval, coalesce window, and connection/HTTP timeouts.
- **Queue tree now communicates state**: each queue's icon, name color, count color, and hover tooltip jointly tell you whether it's empty, normal, backlogged, a dead-letter queue with messages waiting, or whether the broker has refused to disclose its depth. See "Reading the Queue Tree" above.
- **Column widths are remembered across restarts**: once you resize a column in any data table (browser / history / templates / topic messages) the new width persists.
- **Report a Problem**: a new action in the tool window toolbar (next to Settings) and every error notification opens a pre-filled GitHub issue page in your browser. See "Reporting a Problem" below.
- **Duplicate Connection** in plugin settings: clones an existing connection (every field, including credentials and keystore paths) into a fresh dialog with `(copy)` appended to the name - handy for spinning up a near-identical environment without retyping host, port, channel, etc.

## What's New in v1.3.0

- **Disconnect is now truly final**: After clicking Disconnect, background polling and queue count refresh can no longer silently re-establish the connection. The Disconnected state is sticky until you explicitly reconnect.
- **Double-click to reconnect**: Double-clicking a disconnected connection node now reconnects and loads queues, matching the initial-connect behaviour.
- **Right-click no longer triggers browse**: Right-clicking a queue or topic only updates the selection context for the context menu - it no longer switches to the Browser tab or triggers a queue browse.
- **Clean tree reset on disconnect**: The connection node collapses and resets to a "Loading..." placeholder on disconnect so the expand arrow stays visible and reconnect always works on the first try.
- **Failed queue load retries**: A node stuck on "Error: ..." no longer blocks future load attempts - the next expand triggers a fresh retry.
- **IBM MQ queue counts on mTLS and app-only channels**: Channels that do not have PCF admin access now fall back to a direct INQUIRE open, which requires only per-queue BROWSE+INQUIRE rights.

## What's New in v1.2.0 / v1.2.1

- **Message Templates**: Save, reuse, and parameterize frequently sent messages. Templates support variable substitution (`{{uuid}}`, `{{timestamp}}`, custom variables), connection/destination binding, and favorites. See the [Message Templates](#message-templates) section.
- **Template target dialog**: Using a template without a bound connection or destination opens a lightweight dialog that asks only for the missing values at send time.
- **Fixed**: Sending to a sorted paginated queue no longer resets the loaded range or makes new messages disappear.
- **Fixed**: Newly sent messages keep a valid Message ID in the browser via broker send receipts.
- **Fixed**: Artemis move message and topic publish indefinite block.
- **Improved**: Reduced IDE freeze risk - tree search uses cached data, History reads moved off EDT, topic refresh throttled.

## What's New in v1.1.0

- **ActiveMQ Classic 5.x support**: Dedicated connection type with JMS and Jolokia-based management
- **ActiveMQ Classic discovery**: Auto-discovery of queues and topics from broker management MBeans
- **Classic admin actions**: Create/delete queues and topics, queue count, and purge via Jolokia
- **No more blocking dialogs**: Connection test, send, move, and subscription operations no longer freeze the IDE
- **Message body search**: Context-menu find with highlighted matches in Browser, Detail, History, and Send Message
- **Unified body tools**: Consistent Plain / JSON / XML actions and bytes views across all message viewers

## What's New in v1.0.3

- **Artemis topic support**: Auto-discovery, subscribe (durable/non-durable), live message feed
- **Unread badge**: Topics tab shows count of new messages when not active (e.g., "Topics (5)")
- **Publish to topics**: Right-click → Publish to Topic from the connection tree
- **Subscription sync**: Active subscriptions reflected in the tree with indicator, synced in real-time
- **Topic management**: Create/delete topics, manage durable subscriptions via Jolokia

---

## Getting Help

### In the IDE
- **Status bar**: Shows operation status
- **Tooltips**: Hover over fields
- **Context menus**: Right-click for actions

### Reporting a Problem

Two ways to file an issue without leaving the IDE:

1. **Tool window toolbar** → "Report a Problem" (globe icon, next to Settings).
2. **From an error balloon** - every connection-error notification has a "Report this" button that pre-fills the failing connection and error message.

Both open the same dialog where you fill in title, description (Markdown is supported), and an optional contact. Submitting opens a pre-filled GitHub "New Issue" page in your default browser. You sign in to GitHub with your normal browser session to post the issue - the plugin itself does not call any API and does not store any GitHub credentials.

The dialog has an "Include diagnostics" checkbox (on by default) that appends a metadata block to the issue body containing plugin and IDE version, JVM, OS, and how many connections of each type are configured. **Connection names, hosts, and credentials are never included**. There is also a direct link to the issues page if you'd rather skip the form and write the issue manually.

### Online
- **GitHub Issues**: <https://github.com/mdiskuze/message-worker-plugin-issues/issues> - see also the in-IDE "Report a Problem" action above.
- **Changelog**: Version history

---

**Happy message browsing! 🚀**

For questions or issues, please check the plugin's GitHub repository.
