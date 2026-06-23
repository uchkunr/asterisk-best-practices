<p align="center">
  <a href="https://www.asterisk.org/" target="blank">
    <img src="./asterisk_logo.png" width="240" alt="Asterisk Logo" />
  </a>
</p>

<p align="center">A production-grade integration guide and best practices boilerplate for working with <b>Asterisk 23.4.0</b> in Node.js and universal environments.</p>

<p align="center">
  <a href="https://www.asterisk.org/"><img src="https://img.shields.io/badge/asterisk-23.4.0-F37022?logo=asterisk" alt="Asterisk Version" /></a>
</p>

## Overview

Integrating with **Asterisk 23.4.0** requires a deep understanding of its internal communication layers. This guide provides industry-level architectures and design patterns for building robust telephony systems. It covers the core mechanics of inbound calls (**Inbox**), outbound automation (**Outcoming**), queue management (**Queue**), and programmatic call control (**Originate**), derived from real-world, high-traffic systems.

---

## Integration Architecture

Enterprise solutions interact with Asterisk using three main interfaces depending on the requirement:

1. **Asterisk Call Files (High-Throughput Outbound)**
   - **Use Case**: Mass dialing, automated callback campaigns, notification services.
   - **Mechanic**: Text files containing dial instructions are written to a spool folder, where Asterisk processes them natively.
   - **Benefit**: Immune to application network drops or REST API latency during call setup.

2. **Asterisk Manager Interface (AMI - Real-time State & Call Control)**
   - **Use Case**: Active call manipulation, live monitoring, channel spying (ChanSpy), and hangup control.
   - **Mechanic**: Event-based TCP connection using raw commands and response mapping.

3. **Asterisk REST Interface (ARI - Channel Orchestration)**
   - **Use Case**: Building custom interactive applications (IVR, conferences) inside Stasis applications using WebSockets.

---

## 1. Connection Management & Keep-Alive (AMI Heartbeat)

A common pitfall when integrating with Asterisk Manager Interface (AMI) in Node.js is relying solely on library-level reconnect features like `keepConnected()`.

### The Problem with keepConnected()

Many Node.js libraries implement a lazy connection model or reconnect loop which drops TCP sockets during periods of inactivity and reconnects them only when a new action is triggered. This creates significant overhead:

- **Server Overhead**: Floods Asterisk log files with constant authentication and connection handshakes.
- **Latency**: Every new action suffers from TCP handshake and login latency (sometimes taking 100ms+ to authenticate).
- **State Loss**: Inactive sockets miss critical asynchronous events (e.g., operator status changes) because the socket was closed.

### The Solution: Periodic Ping-Pong Heartbeat

To keep a single connection open permanently, you must issue periodic `Ping` actions to the Asterisk AMI socket (e.g., every 30 seconds). This keeps the socket active, prevents TCP timeouts by firewalls, and maintains a clean log history on the Asterisk side.

Reference implementation from `/Users/re4son/projects/work/fetg/callback/src/lib/ami.js`:

```javascript
require("dotenv").config();
const AMI = require("asterisk-manager");

// Initialize AMI manager with persistent socket options
const ami = new AMI(
  process.env.ASTERISK_PORT,
  process.env.ASTERISK_HOST,
  process.env.ASTERISK_USERNAME,
  process.env.ASTERISK_PASSWD,
  true, // keepAlive flag
);

// Keep-Alive heartbeat function sending PING action
const keepConnectedHeartbeat = () => {
  ami.action({ Action: "Ping" });
};

// Send a Ping every 30 seconds to maintain the active TCP pipe
setInterval(keepConnectedHeartbeat, 30000);

module.exports = ami;
```

---

## 2. Outcoming (Call Files Integration)

Using Call Files is the most reliable way to execute asynchronous outbound campaigns.

### Call File Format

Files should be generated using the following standard template:

```text
Channel: Local/998901234567@callbackout
MaxRetries: 1
RetryTime: 60
WaitTime: 30
Context: default
Extension: 998901234567
Priority: 1
Set: sound=default/notification_audio
Set: USERID=998901234567
```

- **Channel**: PJSIP channel or `Local` proxy channels. `Local` channels are preferred because they resolve through the dialplan, allowing advanced routing, failovers, and billing hook insertions.
- **Set**: Direct channel variables. Passing variables allows you to dynamically set playback files or session metadata without duplicating dialplan code.

### Safe Spooling Architecture

Writing files directly to `/var/spool/asterisk/outgoing/` can overload the system and cause race conditions (e.g., Asterisk picking up a partially written file). Implement a two-phase spooler instead:

1. **Staging Directory**: Your Node.js API writes the `.call` file to `/tmp/callfiles/{context}/{timestamp}/phone.call`.
2. **Batch Parameter File**: A companion `param.txt` is created in the same directory:
   ```text
   pause: off
   maxCalls: 5
   ```
3. **Spooler Daemon (`mvtooutgoingdir.sh`)**: A lightweight daemon running as a `systemd` service continuously monitors the staging folder and enforces:
   - **Time Windows**: Blocking execution outside working hours (e.g., 08:00 - 18:00).
   - **Active Pausing**: Checking `pause` states dynamically.
   - **Concurrency Control**: Moving files incrementally matching the `maxCalls` limit to respect network trunks.
   - **Atomic Operation**: Writing files with temporary names and renaming them, or delaying movement until the files are at least 5 seconds old.

---

## 3. PJSIP Endpoint & Contact Monitoring

In Asterisk 23.4.0, the legacy `chan_sip` driver is obsolete. Applications must communicate using `chan_pjsip`. Monitoring SIP registration statuses (softphones, IP-phones, or Trunks) is vital for dashboard visualization and smart call routing.

### Listing All PJSIP Endpoints

Send the `PJSIPShowEndpoints` action via AMI to query the configuration status of PJSIP endpoints:

```text
Action: PJSIPShowEndpoints
```

Asterisk returns a list of `EndpointList` events, completed by an `EndpointListComplete` event.

### Checking Contact Availability & Latency

To check if a dynamic SIP user is registered and measure network round-trip latency, request PJSIP contact details:

```text
Action: PJSIPShowContacts
```

This triggers `ContactList` events, which contain status data (`Available`/`Unavailable`) and round-trip time (`rtt`) metrics for registered SIP phones.

---

## 4. Originate (Programmatic Dialing)

For immediate, user-triggered operations, use the AMI `Originate` action.

### AMI Channel Spying (ChanSpy)

This pattern initiates a call to an operator's channel and immediately links it to a customer's channel for spying or whispering:

```javascript
const initiateSpy = (operatorChannel, targetChannel, mode = "q") => {
  // mode 'w' = whisper (speak to operator), 'q' = quiet spy (listen only)
  ami.action(
    {
      action: "originate",
      channel: `Local/${operatorChannel}@chanspy_context`,
      context: "chanspy",
      application: "ChanSpy",
      Data: `${targetChannel},${mode}`,
      Async: "true",
    },
    (err, resp) => {
      if (err) {
        console.warn(`Spy origination failed: ${err.message}`);
      } else {
        console.info(`Spying established successfully.`);
      }
    },
  );
};
```

### Channel Hangup

To drop active calls programmatically:

```javascript
const forceHangup = (channel) => {
  ami.action(
    {
      action: "hangup",
      channel: channel,
    },
    (err, res) => {
      if (err) {
        console.warn(`Failed to hangup channel ${channel}: ${err.message}`);
      } else {
        console.info(`Channel ${channel} terminated.`);
      }
    },
  );
};
```

---

## 5. Queue Management

To avoid originating calls when no operators are available, the application must monitor queue states.

### Checking Free Agents

Query queue states before scheduling outbound files:

```bash
# Shell check
FREE_OPERATORS=$(asterisk -rx "queue show support" | grep -E "SIP/|PJSIP/" | grep -c "has taken no calls yet" | grep -v "paused")
```

### AMI Queue Status Mapping

Querying queue states programmatically requires sending a `QueueStatus` action and collecting the event stream:

```javascript
const getQueueStatus = async (queueName) => {
  const actionId = `queue_status_${Date.now()}`;
  ami.action({
    action: "QueueStatus",
    Queue: queueName,
    ActionID: actionId,
  });

  const events = await listenEvents(ami, actionId, "QueueStatusComplete");
  return events.filter((e) => e.event === "QueueMember");
};
```

---

## 6. Inbox (Inbound Calls)

Manage inbound traffic cleanly using modular dialplans:

- **Separation of Concerns**: Create separate contexts inside `extensions.conf` for each business path:
  ```ini
  [inbound-callback]
  exten => _X.,1,NoOp(Inbound Callback from ${CALLERID(num)})
  same => n,AGI(agi://localhost:3000/inbound-webhook)
  same => n,Hangup()
  ```
- **FastAGI (Asterisk Gateway Interface)**: Outsource routing decisions to your Node.js application over FastAGI rather than writing complex logic in flat `.conf` files.

---

## 7. Operations & Best Practices

### Preventing Memory Leaks (Event Listeners)

Ensure you remove event listeners after catching the completion token of transactional AMI commands:

```javascript
const listenEvents = (amiInstance, actionId, finalEventName) => {
  return new Promise((resolve) => {
    const list = [];
    const eventHandler = (evt) => {
      if (evt.actionid === actionId) {
        if (evt.event === finalEventName) {
          amiInstance.removeListener("managerevent", eventHandler);
          resolve(list);
        } else {
          list.push(evt);
        }
      }
    };
    amiInstance.on("managerevent", eventHandler);
  });
};
```

### Audio Assets Optimization

- Transcode audio files beforehand to match Asterisk's native telephony rates: **16-bit, 8kHz Mono PCM WAV** (e.g., `slinear` or `ulaw`).
- Avoid file extensions in dialplan/callfile execution: use `sound=default/welcome` instead of `sound=default/welcome.wav`.

### Directory Permissions

Verify folder ownership for spool processes:

```bash
chown -R asterisk:asterisk /var/spool/asterisk/outgoing
chmod -R 755 /var/spool/asterisk/outgoing
```
