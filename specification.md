<h1>OpAMP: Open Agent Management Protocol</h1>

Author: Tigran Najaryan, Splunk

Contributors: Chris Green, Splunk

Status: Early Draft (see Open Questions).

Date: September 2021

Note: address all TODO and Open Questions before considering the document ready for final review.

Note 2: this document requires a simplification pass to reduce the scope, size and complexity.

[TOC]


<h1 id="introduction">Introduction</h1>


Open Agent Management Protocol (OpAMP) is a network protocol for remote
management of large fleets of data collection Agents.

OpAMP allows Agents to report their status to and receive configuration from a
Server and to receive addons and agent installation package updates from the
server. The protocol is vendor-agnostic, so the Server can remotely monitor and
manage a fleet of different Agents that implement OpAMP, including a fleet of
mixed agents from different vendors.

OpAMP supports the following functionality:



* Remote configuration of the agents.
* Status reporting. The protocol allows the agent to report the properties of
  the agent such as its type and version or the operating system type and
  version it runs on. The status reporting also allows the management server to
  tailor the remote configuration to individual agents or types of agents.
* Agent's own telemetry reporting to an
  [OTLP](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/protocol/otlp.md)-compatible
  backend to monitor agent's process metrics such as CPU or RAM usage, as well
  as agent-specific metrics such as rate of data processing.
* Management of downloadable agent-specific addons.
* Secure auto-updating capabilities (both upgrading and downgrading of the
  agents).
* Connection credentials management, including client-side TLS certificate
  revocation and rotation.

The functionality listed above enables a 'single pane of glass' management view
of a large fleet of mixed agents (e.g. OpenTelemetry Collector, Fluentd, etc).

<h1 id="communication-model">Communication Model</h1>


The OpAMP Server manages Agents that implement the client-side of OpAMP
protocol. The Agents can optionally send their own telemetry to an OTLP
destination when directed so by the OpAMP Server. The Agents likely also connect
to other destinations, where they send the data they collect:


```
        ┌────────────┬────────┐           ┌─────────┐
        │            │ OpAMP  │  OpAMP    │ OpAMP   │
        │            │        ├──────────►│         │
        │            │ Client │           │ Server  │
        │            └────────┤           └─────────┘
        │                     │
        │            ┌────────┤           ┌─────────┐
        │            │OTLP    │ OTLP/HTTP │ OTLP    │
        │  Agent     │        ├──────────►│         │
        │            │Exporter│           │ Receiver│
        │            └────────┤           └─────────┘
        │                     │
        │            ┌────────┤
        │            │Other   ├──────────► Other
        │            │Clients ├──────────► Destinations
        └────────────┴────────┘
```


This specification defines the OpAMP network protocol and the expected behavior
for OpAMP Agents and Servers. The OTLP/HTTP protocol is
[specified here](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/protocol/otlp.md).
The protocols used by the Agent to connect to other destinations are agent
type-specific and are outside the scope of this specification.

OpAMP protocol uses [WebSocket](https://datatracker.ietf.org/doc/html/rfc6455)
as the transport. The Agent is a WebSocket client and the Server is a WebSocket
server. The Agent and the Server communicate using binary data WebSocket
messages. The payload of each WebSocket message is a
[binary serialized Protobuf](https://developers.google.com/protocol-buffers/docs/encoding)
message. The Agent sends AgentToServer Protobuf messages and the Server sends
ServerToAgent Protobuf messages:


```
        ┌───────────────┐                        ┌──────────────┐
        │               │      AgentToServer     │              │
        │               ├───────────────────────►│              │
        │     Agent     │                        │    Server    │
        │               │      ServerToAgent     │              │
        │               │◄───────────────────────┤              │
        └───────────────┘                        └──────────────┘
```


Typically a single Server accepts WebSocket connections from many agents. Agents
are identified by self-assigned or server-assigned globally unique instance
identifiers (or instance_uid for short). The instance_uid is recorded in each
message sent from the Agent to the Server and from the Server to the Agent.

The default URL path for the initial WebSocket's HTTP connection is /v1/opamp.
The URL path MAY be configurable on the Agent and on the Server.

OpAMP is an asynchronous, full-duplex message exchange protocol. The order and
sequence of messages exchanged by the Agent and the Server is defined for each
particular capability in the corresponding section of this specification.

The sequence is normally started by an initiating message triggered by some
external event. For example after the connection is established the Agent sends
a StatusReport message. In this case the "connection established" is the
triggering event and the StatusReport is the initiating message.

Both the Agent and the Server may begin a sequence by sending an initiating
message.

The initiating message may trigger the recipient to send one or more messages
back, which in turn may trigger messages in the opposite direction and so on.
This exchange of messages in both directions continues until the sequence is
over because the goal of the exchange is achieved or the sequence failed with an
error.

Note that the same message may in some cases be the initiating message of the
sequence and in some other cases it may be triggered in response to receiving
some other message. Unlike other protocols in OpAMP there is no strict
separation between "request" and "response" messages types. The role of the
message depends on how the sequence is triggered.

For example the StatusReport message may be the initiating message sent by the
Agent when the Agent connects to the Server for the first time. The StatusReport
message may also be sent by the Agent in response to the Server making a remote
configuration offer to the Agent and Agent reporting that it accepted the
configuration.

See sections under the [Operation](#operation) section for the details of the
message sequences.

## AgentToServer Message

The body of the WebSocket message is a binary serialized Protobuf message
AgentToServer as defined below (all messages in this document are specified in
[Protobuf 3 language](https://developers.google.com/protocol-buffers/docs/proto3)):


```protobuf
message AgentToServer {
    string instance_uid = 1;
    StatusReport status_report = 2;
    AgentAddonStatuses addon_statuses = 3;
    AgentInstallStatus agent_install_status = 4;
    AgentDisconnect agent_disconnect = 5;
    AgentToServerFlags flags = 6;
}
```

One or more of the fields (status_report, addon_statuses, agent_install_status,
agent_disconnect) MUST be set. The Server should process each field as it is
described in the corresponding [Operation](#operation) section.

#### instance_uid

The instance_uid field is a globally unique identifier of the running instance
of the Agent. The Agent SHOULD self-generate this identifier and make the best
effort to avoid creating an identifier that may conflict with identifiers
created by other Agents. The instance_uid SHOULD remain unchanged for the
lifetime of the agent process. The recommended format for the instance_uid is
[ULID](https://github.com/ulid/spec).

In case the Agent wants to use an identifier generated by the Server, the field
SHOULD be set with a temporary value and RequestInstanceUid flag MUST be set.

#### status_report

The status of the Agent. MUST be set in the first AgentToServer message that the
Agent sends after connecting. This field SHOULD be unset if this information is
unchanged since the last AgentToServer message for this agent was sent in the
stream.

#### addon_statuses

The list of the agent addons, including addon statuses. This field SHOULD be
unset if this information is unchanged since the last AgentToServer message for
this agent was sent in the stream.

#### agent_install_status

The status of the installation operation that was previously offered by the
server. This field SHOULD be unset if the installation status is unchanged since
the last AgentToServer message.

#### agent_disconnect

AgentDisconnect MUST be set in the last AgentToServer message sent from the
agent to the server.

#### flags


Bit flags as defined by AgentToServerFlags bit masks.

```protobuf
enum AgentToServerFlags {
    FlagsUnspecified = 0;

    // Flags is a bit mask. Values below define individual bits.

    // The agent requests server go generate a new instance_uid, which will
    // be sent back in ServerToAgent message
    RequestInstanceUid     = 0x00000001;
}
```


## ServerToAgent Message

The body of the WebSocket message is a binary serialized Protobuf message
ServerToAgent.

ServerToAgent message is sent from the Server to the Agent either in response to
the AgentToServer message or when the Server has data to deliver to the Agent.

If the Server receives an AgentToServer message and the Server has no data to
send back to the Agent then ServerToAgent message will still be sent, but all
fields except instance_uid will be unset (in that case ServerToAgent serves
simply as an acknowledgement of receipt).

Upon receiving a ServerToAgent message the Agent MUST process it. The processing
that needs to be performed depends on what fields in the message are set. For
details see links to the corresponding sections of this specification from the
field descriptions below.

As a result of this processing the Agent may need to send status reports to the
Server. The Agent is free to perform all the processing of the ServerToAgent
message completely and then send one status report or it may send multiple
status reports as it processes the portions of ServerToAgent message to indicate
the progress (see e.g. [Addon processing](#downloading-addons)). Multiple status
reports may be desirable when processing takes a long time, in which case the
status reports allow the Server to stay informed.

Note that the Server will reply to each status report with a ServerToAgent
message (or with an ServerErrorResponse if something goes wrong). These
ServerToAgent messages may have the same content as the one received earlier or
the content may be different if the situation on the Server has changed. The
Agent SHOULD be ready to process these additional ServerToAgent messages as they
arrive.

The Agent SHOULD NOT send any status reports at all if the status of the Agent
did not change as a result of processing.

The ServerToAgent message has the following structure:

```protobuf
message ServerToAgent {
    string instance_uid = 1;
    ServerErrorResponse error_response = 2;
    AgentRemoteConfig remote_config = 3;
    ConnectionSettingsOffers connection_settings = 4;
    AddonsAvailable addons_available = 5;
    AgentPackageAvailable agent_package_available = 6;
    Flags flags = 7;
    ServerCapabilities capabilities = 8;
    AgentIdentification agent_identification = 9;
    ServerToAgentCommand command = 10;
}
```


#### instance_uid

The Agent instance identifier. MUST match the instance_uid field previously
received in the AgentToServer message. When communication with multiple Agents
is multiplexed into one WebSocket connection (for example when a terminating
proxy is used) the instance_uid field allows to distinguish which Agent the
ServerToAgent message is addressed to.

Note: the value can be overriden by server by sending a new one in the
AgentIdentification field. When this happens then Agent MUST update its
instance_uid to the value provided and use it for all further communication.

#### error_response

error_response is set if the Server wants to indicate that something went wrong
during processing of an AgentToServer message. If error_response is set then all
other fields below must be unset and vice versa, if any of the fields below is
set then error_response must be unset.

#### remote_config


This field is set when the Server has a remote config offer for the Agent. See
[Configuration](#configuration) for details.

#### connection_settings


This field is set when the Server wants the Agent to change one or more of its
client connection settings (destination, headers, certificate, etc). See
[Connection Settings Management](#connection-settings-management) for details.

#### addons_available


This field is set when the Server has addons to offer to the Agent. See
[Addons](#addons) for details.

#### agent_package_available


This field is set when the server has a different version of an agent package
available for download. See [Agent Updates](#agent-package-updates) for details.

#### flags


Bit flags as defined by Flags bit masks.

Report* flags can be used by the server if the agent did not include the
particular bit of information in the last status report (which is an allowed
optimization) but the server does not have it (e.g. was restarted and lost
state).


```protobuf
enum Flags {
    FlagsUnspecified = 0;

    // Flags is a bit mask. Values below define individual bits.

    // The server asks the agent to report effective config. This bit MUST NOT be
    // set if the Agent indicated it cannot report effective config by setting
    // the ReportsEffectiveConfig bit to 0 in StatusReport.capabilities field.
    ReportEffectiveConfig = 0x00000001;

    // The server asks the agent to report addon statuses. This bit MUST NOT be
    // set if the Agent indicated it cannot report addon status by setting
    // the ReportsAddonStatus bit to 0 in StatusReport.capabilities field.
    ReportAddonStatus     = 0x00000002;
}
```

#### capabilities

Bitmask of flags defined by ServerCapabilities enum. All bits that are not
defined in ServerCapabilities enum MUST be set to 0 by the Server. This allows
extending the protocol and the ServerCapabilities enum in the future such that
old Servers automatically report that they don't support the new capability.
This field MUST be set in the first ServerToAgent sent by the Server and MAY be
omitted in subsequent ServerToAgent messages by setting it to
UnspecifiedServerCapability value.

```protobuf
enum ServerCapabilities {
    // The capabilities field is unspecified.
    UnspecifiedServerCapability = 0;
    // The Server can accept status reports. This bit MUST be set, since all Server
    // MUST be able to accept status reports.
    AcceptsStatus                  = 0x00000001;
    // The Server can offer remote configuration to the Agent.
    OffersRemoteConfig             = 0x00000002;
    // The Server can accept EffectiveConfig in StatusReport.
    AcceptsEffectiveConfig         = 0x00000004;
    // The Server can offer Addons.
    OffersAddons                   = 0x00000008;
    // The Server can accept Addon status.
    AcceptsAddonsStatus            = 0x00000010;
    // The Server can offer packages to install.
    OffersAgentPackage             = 0x00000020;
    // The Server can accept the installation status of the package.
    AcceptsAgentPackageStatus      = 0x00000040;
    // The Server can offer connection settings.
    OffersConnectionSettings       = 0x00000080;

    // Add new capabilities here, continuing with the least significant unused bit.
}
```


#### agent_identification

Properties related to identification of the agent, which can be overriden by the
server if needed. When new_instance_uid is set, Agent MUST update instance_uid
to the value provided and use it for all further communication.

```protobuf
message AgentIdentification {
  string new_instance_uid = 1;
}
```

#### command

This field is set when the server wants the agent to
perform a restart. This field must not be set with other fields
besides instance_uid or capabilities. All other fields will be ignored and the
agent will execute the command. See [ServerToAgentCommand Message](#servertoagentcommand-message)
for details.

<h2 id="servererrorresponse-message">ServerErrorResponse Message</h2>


The message has the following structure:


```protobuf
message ServerErrorResponse {
    enum Type {
        UNKNOWN = 0;
        BAD_REQUEST = 1;
        UNAVAILABLE = 2
    }
    Type type = 1;
    string error_message = 2;
    oneof Details {
        RetryInfo retry_info = 3;
    }
}
```



<h4 id="type">type</h4>


This field defines the type of the error that the Server encountered when trying
to process the Agent's request. Possible values are:

UNKNOWN: Unknown error. Something went wrong, but it is not known what exactly.
The error_message field may contain a description of the problem.

BAD_REQUEST: Only sent as a response to a previously received AgentToServer
message and indicates that the AgentToServer message was malformed. See
[Bad Request](#bad-request) processing.

UNAVAILABLE: The server is overloaded and unable to process the request. See
[Throttling](#throttling).

<h4 id="error_message">error_message</h4>


Error message, typically human readable.

<h4 id="retry_info">retry_info</h4>


Additional [RetryInfo](#throttling) message about retrying if type==UNAVAILABLE.

<h2 id="servertoagentcommand-message">ServerToAgentCommand Message</h2>
 

The message has the following structure:

```protobuf
// ServerToAgentCommand is sent from the server to the agent to request that the agent
// perform a command.
message ServerToAgentCommand {
    enum CommandType {
        // The agent should restart. This request will be ignored if the agent does not
        // support restart.
        Restart = 0;
    }
    CommandType type = 1;
}
```

The ServerToAgentCommand message is sent when the Server wants the Agent to restart.
This message must only contain the command, instance_uid, and capabilities fields.  All other fields
will be ignored.

# Operation


<h2 id="status-reporting">Status Reporting</h2>


The Agent MUST send a status report:



* First time immediately after connecting to the Server. The status report MUST
  be the first message sent by the Agent.
* Subsequently every time the status of the Agent changes.

The status report is sent as an AgentToServer message, where the [Body](#body)
field is set to [StatusReport](#statusreport-message) message.

The Server MUST respond to the status report by sending a
[ServerToAgent](#servertoagent-message) message.

If the status report processing failed then the
[error_response](#error_response) field MUST be set to ServerErrorResponse
message.

If the status report is processed successfully by the Server then the
[error_response](#error_response) field MUST be unset and the other fields can
be populated as necessary.

Here is the sequence diagram that shows how status reporting works (assuming
server-side processing is successful):


```
        Agent                                  Server

          │                                       │
          │                                       │
          │          WebSocket Connect            │
          ├──────────────────────────────────────►│
          │                                       │
          │     AgentToServer{StatusReport}       │   ┌─────────┐
          ├──────────────────────────────────────►├──►│         │
          │                                       │   │ Process │
          │           ServerToAgent{}             │   │ Status  │
          │◄──────────────────────────────────────┤◄──┤         │
          │                                       │   └─────────┘
          .                 ...                   .

          │     AgentToServer{StatusReport}       │   ┌─────────┐
          ├──────────────────────────────────────►├──►│         │
          │                                       │   │ Process │
          │           ServerToAgent{}             │   │ Status  │
          │◄──────────────────────────────────────┤◄──┤         │
          │                                       │   └─────────┘
          │                                       │
```


Note that the status of the Agent may change as a result of receiving a message
from the Server. For example the Server may send a remote configuration to the
Agent. Once the Agent processes such a request the Agent's status changes (e.g.
the effective configuration of the Agent changes). Such status change should
result in the Agent sending a status report to the Server.

So, essentially in such cases the sequence of messages may look like this:


```
                   Agent                                  Server

                    │         ServerToAgent{}               │
            ┌───────┤◄──────────────────────────────────────┤
            │       │                                       │
            ▼       │                                       │
        ┌────────┐  │                                       │
        │Process │  │                                       │
        │Received│  │                                       │
        │Data    │  │                                       │
        └───┬────┘  │                                       │
            │       │                                       │
            │Status │                                       │
            │Changed│   AgentToServer{StatusReport}         │   ┌─────────┐
            └──────►├──────────────────────────────────────►├──►│         │
                    │                                       │   │ Process │
                    │         ServerToAgent{}               │   │ Status  │
                    │◄──────────────────────────────────────┤◄──┤         │
                    │                                       │   └─────────┘
```


When the Agent receives a ServerToAgent message the Agent MUST NOT send a status
report unless processing of the message received from the Server resulted in
actual change of the Agent status (e.g. the configuration of the Agent has
changed). The sequence diagram in this case look like this:


```
                     Agent                                  Server

                       │         ServerToAgent{}               │
                ┌──────┤◄──────────────────────────────────────┤
                │      │                                       │
                ▼      │                                       │
            ┌────────┐ │                                       │
            │Process │ │                                       │
            │Received│ │                                       │
            │Data    │ │                                       │
            └───┬────┘ │                                       │
                │      │                                       │
                ▼      │                                       │
             No Status │                                       │
              Changes  │                                       │
                       │                                       │
                       │                                       │
```


Important: if the Agent does not follow these rules the operation may result in
an infinite loop of messages sent back and forth between the Agent and the
Server. TODO: add a section explaining how infinite oscillations between remote
config and status reporting are possible if an attribute is reported in the
status that can be changed via remote config and how to prevent it.

<h3 id="statusreport-message">StatusReport Message</h3>


StatusReport message has the following structure:


```protobuf
message StatusReport {
    AgentDescription agent_description = 1;
    EffectiveConfig effective_config = 2;
    RemoteConfigStatus remote_config_status = 3;
    AgentCapabilities capabilities = 4;
}
```


<h4 id="agent_description">agent_description</h4>


The description of the agent, its type, where it runs, etc. See
[AgentDescription](#agentdescription-message) message for details.

This field SHOULD be unset if no Agent description fields have changed since the
last StatusReport was sent.

<h4 id="effective_config">effective_config</h4>


The current effective configuration of the Agent. The effective configuration is
the one that is currently used by the Agent. The effective configuration may be
different from the remote configuration received from the Server earlier, e.g.
because the agent uses a local configuration instead (or in addition). See
[EffectiveConfig](#effectiveconfig-message) message for details.

This field SHOULD be unset if the effective configuration has not changed since
the last StatusReport message was sent.

<h4 id="remote_config_status">remote_config_status</h4>


The status of the remote config that was previously received from the server.
See [RemoteConfigStatus](#remoteconfigstatus-message) message for details.

This field SHOULD be unset if the remote config status is unchanged since the
last StatusReport message was sent.

#### capabilities

Bitmask of flags defined by AgentCapabilities enum. All bits that are not
defined in AgentCapabilities enum MUST be set to 0 by the Agent. This allows
extending the protocol and the AgentCapabilities enum in the future such that
old Agents automatically report that they don't support the new capability. This
field MUST be set in the first StatusReport sent by the Agent and MAY be omitted
in subsequent StatusReport messages by setting it to UnspecifiedAgentCapability
value.

```protobuf
enum AgentCapabilities {
    // The capabilities field is unspecified.
    UnspecifiedAgentCapability = 0;
    // The Agent can report status. This bit MUST be set, since all Agents MUST
    // report status.
    ReportsStatus                  = 0x00000001;
    // The Agent can accept remote configuration from the Server.
    AcceptsRemoteConfig            = 0x00000002;
    // The Agent will report EffectiveConfig in StatusReport.
    ReportsEffectiveConfig         = 0x00000004;
    // The Agent can accept Addon offers.
    AcceptsAddons                  = 0x00000008;
    // The Agent can report Addon status.
    ReportsAddonsStatus            = 0x00000010;
    // The Agent can accept packages to install.
    AcceptsAgentPackage            = 0x00000020;
    // The Agent can report the installation status of the package.
    ReportsAgentPackageStatus      = 0x00000040;
    // The Agent can report own traces to the destination specified by
    // the Server via ConnectionSettingsOffers.own_traces field.
    ReportsOwnTraces               = 0x00000080;
    // The Agent can report own metrics to the destination specified by
    // the Server via ConnectionSettingsOffers.own_metrics field.
    ReportsOwnMetrics              = 0x00000100;
    // The Agent can report own logs to the destination specified by
    // the Server via ConnectionSettingsOffers.own_logs field.
    ReportsOwnLogs                 = 0x00000200;
    // The can accept connections settings for OpAMP via
    // ConnectionSettingsOffers.opamp field.
    AcceptsOpAMPConnectionSettings = 0x00000400;
    // The can accept connections settings for other destinations via
    // ConnectionSettingsOffers.other_connections field.
    AcceptsOtherConnectionSettings = 0x00000800;
    // The Agent can accept restart requests.
    AcceptsRestartCommand         = 0x00001000;

    // Add new capabilities here, continuing with the least significant unused bit.
}
```

<h3 id="agentdescription-message">AgentDescription Message</h3>

The AgentDescription message has the following structure:

```protobuf
message AgentDescription {
    repeated KeyValue identifying_attributes = 1;
    repeated KeyValue non_identifying_attributes = 2;
}
```

#### identifying_attributes

Attributes that identify the agent.

Keys/values are according to OpenTelemetry semantic conventions, see:
https://github.com/open-telemetry/opentelemetry-specification/tree/main/specification/resource/semantic_conventions

For standalone running Agents (such as OpenTelemetry Collector) the following
attributes SHOULD be specified:

- service.name should be set to a reverse FQDN that uniquely identifies the
  agent type, e.g. "io.opentelemetry.collector"
- service.namespace if it is used in the environment where the Agent runs.
- service.version should be set to version number of the Agent build.
- service.instance.id should be set. It may be be set equal to the Agent's
  instance uid (equal to ServerToAgent.instance_uid field) or any other value
  that uniquely identifies the Agent in combination with other attributes.
- any other attributes that are necessary for uniquely identifying the Agent's
  own telemetry.

The Agent SHOULD also include these attributes in the Resource of its own
telemetry. The combination of identifying attributes SHOULD be sufficient to
uniquely identify the Agent's own telemetry in the destination system to which
the Agent sends its own telemetry.

#### non_identifying_attributes

Attributes that do not necessarily identify the Agent but help describe where it
runs.

The following attributes SHOULD be included:

- os.type, os.version - to describe where the agent runs.
- host.* to describe the host the agent runs on.
- cloud.* to describe the cloud where the host is located.
- any other relevant Resource attributes that describe this agent and the
  environment it runs in.
- any user-defined attributes that the end user would like to associate with
  this agent.

<h3 id="effectiveconfig-message">EffectiveConfig Message</h3>


The EffectiveConfig message has the following structure:


```protobuf
message EffectiveConfig {
    bytes hash = 1;
    AgentConfigMap config_map = 2;
}
```


<h4 id="hash">hash</h4>

The hash of the effective config. The hash is calculated by the Agent from the
content of the effective config (from the names and content of all items in the
config_map field).

After establishing the OpAMP connection if the effective config did not change
since it was last reported during the previous connection sessions the Agent is
recommended to include only the hash field and omit the config_map field to save
bandwidth.

The Server SHOULD compare this hash with the last hash of effective config it
received from the Agent and if the hashes are different the Server SHOULD ask
the Agent to report its full effective config by sending a ServerToAgent message
with ReportEffectiveConfig flag set.

<h4 id="config_map">config_map</h4>


The effective config of the Agent. SHOULD be omitted if unchanged since last
reported.

MUST be set if the Agent has received the ReportEffectiveConfig flag in the
ServerToAgent message.

See AgentConfigMap message definition in the [Configuration](#configuration)
section.

<h3 id="remoteconfigstatus-message">RemoteConfigStatus Message</h3>


The RemoteConfigStatus message has the following structure:


```protobuf
message RemoteConfigStatus {
    bytes last_remote_config_hash = 1;
    enum Status {
        // Remote config was successfully applied by the Agent.
        APPLIED = 0;

        // Agent is currently applying the remote config that it received earlier.
        APPLYING = 1;

        // Agent tried to apply the config received earlier, but it failed.
        // See error_message for more details.
        FAILED = 2;
    }
    Status status = 2;
    string error_message = 3;
}
```


<h4 id="last_remote_config_hash">last_remote_config_hash</h4>


The hash of the remote config that was last received by this agent from the
management server. The server SHOULD compare this hash with the config hash it
has for the agent and if the hashes are different the server MUST include the
remote_config field in the response in the ServerToAgent message.

<h4 id="status">status</h4>


The status of the Agent's attempt to apply a previously received remote
configuration.

<h4 id="error_message">error_message</h4>


Optional error message if status==FAILED.

<h3 id="agentaddonstatuses-message">AgentAddonStatuses Message</h3>


The AgentAddonStatuses message describes the status of all addons that the agent
has or was offered. The message has the following structure:


```protobuf
message AgentAddonStatuses {
    map<string, AgentAddonStatus> addons = 1;
    bytes server_provided_all_addons_hash = 2;
}
```


<h4 id="addons">addons</h4>


A map of AgentAddonStatus messages, where the keys are addon names. The key MUST
match the name field of [AgentAddonStatus](#agentaddonstatus-message) message.

<h4 id="server_provided_all_addons_hash">server_provided_all_addons_hash</h4>


The aggregate hash of all addons that this Agent previously received from the
server via AddonsAvailable message.

The server SHOULD compare this hash to the aggregate hash of all addons that it
has for this Agent and if the hashes are different the server SHOULD send an
AddonsAvailable message to the agent.

<h3 id="agentaddonstatus-message">AgentAddonStatus Message</h3>


The AgentAddonStatus has the following structure:


```protobuf
message AgentAddonStatus {
    string name = 1;
    string agent_has_version = 2;
    bytes agent_has_hash = 3;
    string server_offered_version = 4;
    bytes server_offered_hash = 5;
    enum Status {
        INSTALLED = 0;
        INSTALLING = 1;
        INSTALL_FAILED = 2;
    }
    Status status = 6;
    string error_message = 7;
}
```


<h4 id="name">name</h4>


Addon name. MUST be always set and MUST match the key in the addons field of
AgentAddonStatuses message.

#### agent_has_version

The version of the addon that the Agent has.

MUST be set if the Agent has this addon.

MUST be empty if the Agent does not have this addon. This may be the case for
example if the addon was offered by the Server but failed to install and the
agent did not have this addon previously.

#### agent_has_hash

The hash of the addon that the Agent has.

MUST be set if the Agent has this addon.

MUST be empty if the Agent does not have this addon. This may be the case for
example if the addon was offered by the Server but failed to install and the
agent did not have this addon previously.

#### server_offered_version

The version of the addon that the server offered to the agent.

MUST be set if the installation of the addon is initiated by an earlier offer
from the server to install this addon.

MUST be empty if the Agent has this addon but it was installed locally and was
not offered by the server.

Note that it is possible for both agent_has_version and server_offered_version
fields to be set and to have different values. This is for example possible if
the agent already has a version of the addon successfully installed, the server
offers a different version, but the agent fails to install that version.

#### server_offered_hash

The hash of the addon that the server offered to the agent.

MUST be set if the installation of the addon is initiated by an earlier offer
from the server to install this addon.

MUST be empty if the Agent has this addon but it was installed locally and was
not offered by the server.

Note that it is possible for both agent_has_hash and server_offered_hash fields
to be set and to have different values. This is for example possible if the
agent already has a version of the addon successfully installed, the server
offers a different version, but the agent fails to install that version.

<h4 id="status">status</h4>


The status of this addon. The possible values are:

INSTALLED: Addon is successfully installed by the Agent. The error_message field
MUST NOT be set.

INSTALLING: Agent is currently downloading and installing the addon.
server_offered_hash field MUST be set to indicate the version that the agent is
installing. The error_message field MUST NOT be set.

INSTALL_FAILED: Agent tried to install the addon but installation failed.
server_offered_hash field MUST be set to indicate the version that the agent
tried to install. The error_message may also contain more details about the
failure.

<h4 id="error_message">error_message</h4>


An error message if the status is erroneous.

<h3 id="agentinstallstatus-message">AgentInstallStatus Message</h3>


This message contains the status of the last agent package install status
performed by the agent and has the following structure:


```protobuf
message AgentInstallStatus {
    string server_offered_version = 1;
    bytes server_offered_hash = 2;
    enum Status {
        INSTALLED = 0;
        INSTALLING = 1;
        INSTALL_FAILED = 2;
        INSTALL_NO_PERMISSION = 3;
    }
    Status status = 3;
    string error_message = 4;
}
```

<h4 id="server_offered_version">server_offered_version</h4>

The version field from the AgentPackageAvailable that the server offered to the
agent. MUST be set if the agent previously received an offer from the server to
install this agent.


<h4 id="server_offered_hash">server_offered_hash</h4>


The hash of the agent package file that the server offered to the agent. MUST be
set if the agent previously received an offer from the server to install this
agent.

<h4 id="status">status</h4>


The status of the agent package installation operation. The possible values are:

INSTALLED: Agent package was successfully installed. error_message MUST NOT be
set.

INSTALLING: Agent is currently downloading and installing the package.
server_offered_hash MUST be set to indicate the version that the agent is
installing. error_message MUST NOT be set.

INSTALL_FAILED: Agent tried to install the package but installation failed.
server_offered_hash MUST be set to indicate the package that the agent tried to
install. error_message may also contain more details about the failure.

INSTALL_NO_PERMISSION: Agent did not install the package because it is not
permitted to. This may be for example the case when operating system permissions
prevent the agent from self-updating or when self-updating is disabled by the
user. error_message may also contain more details about what exactly is not
permitted.

<h4 id="error_message">error_message</h4>


Optional human readable error message if the status is erroneous.

<h2 id="connection-settings-management">Connection Settings Management</h2>


OpAMP includes features that allow the Server to manage Agent's connection
settings for all of the destinations that the agent connects to.

The following diagram shows a typical Agent that is managed by OpAMP Servers,
sends its own telemetry to an OTLP backend and also connects to other
destinations to perform its work:


```
            ┌────────────┬────────┐           ┌─────────┐
            │            │ OpAMP  │  OpAMP    │ OpAMP   │
            │            │        ├──────────►│         │
            │            │ Client │           │ Server  │
            │            └────────┤           └─────────┘
            │                     │
            │            ┌────────┤           ┌─────────┐
            │            │OTLP    │ OTLP/HTTP │OTLP     │
            │  Agent     │        ├──────────►│Telemetry│
            │            │Client  │           │Backend  │
            │            └────────┤           └─────────┘
            │                     │
            │            ┌────────┤
            │            │Other   ├──────────► Other
            │            │        ├──────────► 
            │            │Clients ├──────────► Destinations
            └────────────┴────────┘
```


When connecting to the OpAMP Server and to other destinations it is typically
expected that Agents will use some sort of header-based authorization mechanism
(e.g. an "Authorization" HTTP header or an access token in a custom header) and
optionally also client-side certificates for TLS connections (also known as
mutual TLS).

OpAMP protocol allows the Server to offer settings for each of these connections
and for the Agent to accept or reject such offers. This mechanism can be used to
direct the Agent to a specific destination, as well as for access token and TLS
certificate registration, revocation and rotation as needed.

The Server can offer connection settings for the following 3 classes of
destinations:



1. The **OpAMP Server** itself. This is typically used to manage credentials
   such as the TLS certificate or the request headers that are used for
   authorization. The Server MAY also offer a different destination endpoint to
   direct the Agent to connect to a different OpAMP Server.
2. The destinations for the Agent to send its **own telemetry**: metrics, traces
   and logs using OTLP/HTTP protocol.
3. A set of **additional "other" connection** settings, with a string name
   associated with each. How the agent type uses these is agent-specific.
   Typically the name represents the name of the destination to connect to (as
   it is known to the agent). For example OpenTelemetry Collector can use the
   named connection settings for its exporters, one named connection setting per
   correspondingly named exporter.

The Server may make an offer for the particular connection class only if the
corresponding capability to use the connection is reported by the Agent via
StatusReport.capabilities field:

- If ReportsOwnTraces capability bit is set the Server may offer connection
  settings for traces using own_traces field.
- If ReportsOwnMetrics capability bit is set the Server may offer connection
  settings for metrics using own_metrics field.
- If ReportsOwnLogs capability bit is set the Server may offer connection
  settings for logs using own_logs field.
- If AcceptsOpAMPConnectionSettings capability bit is set the Server may offer
  connection settings for OpAMP connection using opamp field.
- If AcceptsOtherConnectionSettings capability bit is set the Server may offer
  connection settings for other destinations using other_connections field.

Depending on which connection settings are offered the sequence of operations is
slightly different. The handling of connection settings for own telemetry is
described in [Own Telemetry Reporting](#own-telemetry-reporting). The handling
of connection settings for "other" destinations is described in
[Connection Settings for "Other" Destinations](#connection-settings-for-"other"-destinations).
The handling of OpAMP connection settings is described below.

<h3 id="opamp-connection-setting-offer-flow">OpAMP Connection Setting Offer Flow</h3>


Here is how the OpAMP connection settings change happens:


```
                   Agent                                 Server

                     │                                       │    Initiate
                     │    Connect                            │    Settings
                     ├──────────────────────────────────────►│     Change
                     │                 ...                   │        │
                     │                                       │◄───────┘
                     │                                       │          ┌───────────┐
                     │                                       ├─────────►│           │
                     │                                       │ Generate │Credentials│
┌───────────┐        │ServerToAgent{ConnectionSettingsOffers}│ and Save │   Store   │
│           │◄───────┤◄──────────────────────────────────────┤◄─────────┤           │
│Credentials│ Save   │                                       │          └───────────┘
│   Store   │        │             Disconnect                │
│           ├───────►├──────────────────────────────────────►│
└───────────┘        │                                       │
                     │    Connect, New settings              │          ┌───────────┐
                     ├──────────────────────────────────────►├─────────►│           │
                     │                                       │ Delete   │Credentials│
┌───────────┐        │    Connection established             │ old      │   Store   │
│           │◄───────┤◄─────────────────────────────────────►│◄─────────┤           │
│Credentials│Delete  │                                       │          └───────────┘
│   Store   │old     │                                       │
│           ├───────►│                                       │
└───────────┘        │                                       │

```



1. Server generates new connection settings and saves it in Server's credentials
   store, associating the new settings with the Agent instance UID.
2. Server sends the ServerToAgent message that includes
   [ConnectionSettingsOffers](#connectionsettingsoffers-message) message. The
   [opamp](#opamp) field contains the new
   [ConnectionSettings](#connectionsettings-message) offered. The server sets
   only the fields that it wants to change in the
   [ConnectionSettings](#connectionsettings-message) message. The server can
   offer to replace a single field (e.g. only the [headers](#headers)) or
   several of the fields at once.
3. Agent receives the settings offer and updates the fields that the offer
   includes on top of its current connection settings, then saves the updated
   connection settings in the local store, marking it as "candidate" (if Agent
   crashes it will retry "candidate" validation steps 5-9).
4. Agent disconnects from the Server.
5. Agent connects to the Server, using the new settings.
6. Connection is successfully established, any TLS verifications required are
   passed and the server indicates a successful authorization.
7. Server deletes the old connection settings for this agent (using Agent
   instance UID) from its credentials store.
8. Agent deletes the old settings from its credentials store and marks the new
   connection settings as "valid".
9. If step 6 fails the Agent deletes the new settings and reverts to the old
   settings and reconnects.

Note: Agents which are unable to persist new connection settings and have access
only to ephemeral storage SHOULD reject certificate offers otherwise they risk
losing access after restarting and losing the offered certificate.

<h3 id="trust-on-first-use">Trust On First Use</h3>


Agents that want to use TLS with a client certificate but do not initially have
a certificate can use the Trust On First Use (TOFU) flow. The sequence is the
following:



* Agent connects to the Server using regular TLS (validating Server's identity)
  but without a client certificate. Agent sends its Status Report so that it can
  be identified.
* The Server accepts the connection and status and awaits for an approval to
  generate a client certificate for the Agent.
* Server either waits for a manual approval by a human or automatically approves
  all TOFU requests if the Server is configured to do so (can be a server-side
  option).
* Once approved the flow is essentially identical to
  [OpAMP Connection Setting Offer Flow](#opamp-connection-setting-offer-flow)
  steps, except that there is no old client certificate to delete.

TOFU flow allows to bootstrap a secure environment without the need to perform
Agent-side installation of certificates.

Exact same TOFU approach can be also used for Agents that don't have the
necessary authorization headers to access the Server. The Server can detect such
access and upon approval send the authorization headers to the Agent.

<h3 id="registration-on-first-use">Registration On First Use</h3>


In some use cases it may be desirable to equip newly installed Agents with an
initial connection settings that are good for the first use, but generate a new
set of connection credentials after the first connection is established.

This can be achieved very similarly to how the TOFU flow works. The only
difference is that the first connection will be properly authenticated, but the
Server will immediately generate and offer new connection settings to the Agent.
The Agent will then persist the setting and will use them for all subsequent
operations.

This allows deploying a large number of Agents using one pre-defined set of
connection credentials (authorization headers, certificates, etc), but
immediately after successful connection each Agent will acquire their own unique
connection credentials. This way individual Agent's credentials may be revoked
without disrupting the access to all other Agents.

<h3 id="revoking-access">Revoking Access</h3>


Since the Server knows what access headers and a client certificate the Agent
uses, the Server can revoke access to individual Agents by marking the
corresponding connection settings as "revoked" and disconnecting the Agent.
Subsequent connections using the revoked credentials can be rejected by the
Server essentially prohibiting the Agent to access the Server.

Since the Server has control over the connection settings of all 3 destination
types of the Agent (because it can offer the connection settings) this
revocation may be performed for either of the 3 types of the destinations,
provided that the Server previously offered and the Agent accepted the
particular type of destination.

For own telemetry and "other" destinations the Server MUST also communicate the
revocation fact to the corresponding destinations so that they can begin
rejecting access to connections that use the revoked credentials.

<h3 id="certificate-generation">Certificate Generation</h3>


Client certificates that the Server generates may be self-signed, signed by a
private Certificate Authority or signed by a public Certificate Authority. The
Server is responsible for generating client certificates such that they are
trusted by the destination the certificate is intended for. This requires that
either the destinations remember and trust the individual self-signed client
certificate's public key directly or they trust the Certificate Authority that
is used for signing the client certificate so that the trust chain can be
verified.

How exactly the client certificates are generated is outside the scope of the
OpAMP specification.

<h3 id="connection-settings-for-"other"-destinations">Connection Settings for "Other" Destinations</h3>


TBD

<h3 id="connectionsettingsoffers-message">ConnectionSettingsOffers Message</h3>


ConnectionSettingsOffers message describes connection settings for the agent to
use:


```protobuf
message ConnectionSettingsOffers {
    bytes hash = 1;
    ConnectionSettings opamp = 2;
    ConnectionSettings own_metrics = 3;
    ConnectionSettings own_traces = 4;
    ConnectionSettings own_logs = 5;
    map<string,ConnectionSettings> other_connections = 6;
}
```


#### hash

Hash of all settings, including settings that may be omitted from this message
because they are unchanged.

#### opamp

Settings to connect to the OpAMP server. If this field is not set then the agent
should assume that the settings are unchanged and should continue using existing
settings. The agent MUST verify the offered connection settings by actually
connecting before accepting the setting to ensure it does not lose access to
the OpAMP server due to invalid settings.

#### own_metrics

Settings to connect to an OTLP metrics backend to send agent's own metrics to.
If this field is not set then the agent should assume that the settings are
unchanged.

#### own_traces

Settings to connect to an OTLP metrics backend to send agent's own traces to. If
this field is not set then the agent should assume that the settings are
unchanged.

#### own_logs

Settings to connect to an OTLP metrics backend to send agent's own logs to. If
this field is not set then the agent should assume that the settings are
unchanged.

#### other_connections

Another set of connection settings, with a string name associated with each. How
the agent uses these is agent-specific. Typically the name represents the name
of the destination to connect to (as it is known to the agent). If this field is
not set then the agent should assume that the other_connections settings are
unchanged.

<h3 id="connectionsettings-message">ConnectionSettings Message</h3>


ConnectionSettings describes connection settings for one destination. The
message has the following structure:


```protobuf
message ConnectionSettings {
    string destination_endpoint = 1;
    Headers headers = 2;
    string proxy_endpoint = 3;
    Headers proxy_headers = 4;
    TLSCertificate certificate = 5;
    enum Flags {
        _ = 0;
        DestinationEndpointSet = 0x01;
        ProxyEndpointSet = 0x02;
    }
    Flags flags = 6;
}
```


<h4 id="destination_endpoint">destination_endpoint</h4>


A URL, host:port or some other destination specifier.

For OpAMP destination this MUST be a WebSocket URL and MUST be non-empty, for
example: "wss://example.com:4318/v1/opamp"

For Agent's own telemetry destination this MUST be the full HTTP URL to an
OTLP/HTTP/Protobuf receiver. The value MUST be a full URL with path and schema
and SHOULD begin with "https://", for example
"[https://example.com:4318/v1/metrics](https://example.com:4318/v1/metrics)".
The Agent MAY refuse to send the telemetry if the URL begins with "http://".

The field is considered unset if (flags & DestinationEndpointSet)==0.

<h4 id="headers">headers</h4>


Headers to use when connecting. Typically used to set access tokens or other
authorization headers. For HTTP-based protocols the agent should set these in
the request headers.

For example:

key="Authorization", Value="Basic YWxhZGRpbjpvcGVuc2VzYW1l".

if the field is unset then the agent SHOULD continue using the headers that it
currently has (if any).

<h4 id="proxy_endpoint">proxy_endpoint</h4>


A URL, host:port or some other specifier of an intermediary proxy. Empty if no
proxy is used.

Example use case: if OpAMP proxy is also an OTLP intermediary Collector then the
OpAMP proxy can direct the Agents that connect to it to also send Agents's OTLP
metrics through its OTLP metrics pipeline. Can be used for example by
OpenTelemetry Helm chart with 2 stage-collection when Agents on K8s nodes are
proxied through a standalone Collector.

For example: "https://proxy.example.com:5678"

The field is considered unset if (flags & ProxyEndpointSet)==0.

<h4 id="proxy_headers">proxy_headers</h4>


Headers to use when connecting to a proxy. For HTTP-based protocols the agent
should set these in the request headers. If no proxy is used the Headers field
must be present and must contain no headers.

For example:

key="Proxy-Authorization", value="Basic YWxhZGRpbjpvcGVuc2VzYW1l".

if the field is unset then the agent SHOULD continue using the proxy headers
that it currently has (if any).

<h4 id="certificate">certificate</h4>


The agent should use the offered certificate to connect to the destination from
now on. If the agent is able to validate and connect using the offered
certificate the agent SHOULD forget any previous client certificates for this
connection.

This field is used to perform a client certificate revocation/rotation. if the
field is unset then the agent SHOULD continue using the certificate that it
currently has (if any).

<h4 id="flags">flags</h4>


​​Bitfield of Flags enum:


```
enum Flags {
    _ = 0;
    DestinationEndpointSet = 0x01;
    ProxyEndpointSet = 0x02;
}
```


<h3 id="headers-message">Headers Message</h3>



```
message Headers {
    repeated Header headers = 1;
}
message Header {
    string key = 1;
    string value = 2;
}
```


<h3 id="tlscertificate-message">TLSCertificate Message</h3>


The message carries a TLS certificate that can be used as a client-side
certificate.

The (public_key,private_key) certificate pair should be issued and signed by a
Certificate Authority that the destination server recognizes.

Alternatively the certificate may be self-signed, assuming the server can verify
the certificate. In this case the ca_public_key field can be omitted.


```protobuf
message TLSCertificate {
    bytes public_key = 1;
    bytes private_key = 2;
    bytes ca_public_key = 3;
}
```


<h4 id="public_key">public_key</h4>


PEM-encoded public key of the certificate. Required.

<h4 id="private_key">private_key</h4>


PEM-encoded private key of the certificate. Required.

<h4 id="ca_public_key">ca_public_key</h4>


PEM-encoded public key of the CA that signed this certificate. Optional, MUST be
specified if the certificate is CA-signed. Can be stored by intermediary
TLS-terminating proxies in order to verify the connecting client's certificate
in the future.

<h2 id="own-telemetry-reporting">Own Telemetry Reporting</h2>


Own Telemetry Reporting is an optional capability of OpAMP protocol. The Server
can offer to the Agent a destination to which the Agent can send its own
telemetry (metrics, traces or logs). If the Agent is capable of producing
telemetry and wishes to do so then it should sends its telemetry to the offered
destination using OTLP/HTTP protocol:


```
            ┌────────────┬────────┐           ┌─────────┐
            │            │ OpAMP  │  OpAMP    │ OpAMP   │
            │            │        ├──────────►│         │
            │            │ Client │           │ Server  │
            │            └────────┤           └─────────┘
            │   Agent             │
            │            ┌────────┤           ┌─────────┐
            │            │OTLP    │ OTLP/HTTP │ OTLP    │
            │            │        ├──────────►│ Metric  │
            │            │Exporter│           │ Backend │
            └────────────┴────────┘           └─────────┘
```


The Server makes the offer by sending a [ServerToAgent](#servertoagent-message)
message with a populated [connection_settings](#connection_settings) field that
contains one or more of the own_metrics, own_traces, own_logs fields set. Each
of these fields describes a destination, which can receive telemetry using OTLP
protocol.

The Server SHOULD populate the [connection_settings](#connection_settings) field
when it sends the first ServerToAgent message to the particular Agent (normally
in response to the first status report from the Agent), unless there is no OTLP
backend that can be used. The Server SHOULD also populate the field on
subsequent ServerToAgent if the destination has changed. If the destination is
unchanged the connection_settings field SHOULD NOT be set. When the Agent
receives a ServerToAgent with an unset connection_settings field the Agent SHOULD
continue sending its telemetry to the previously offered destination.

The Agent SHOULD periodically report its metrics to the destination offered in
the [own_metrics](#own_metrics) field. The recommended reporting interval is 10
seconds. Here is the diagram that shows the operation sequence:


```
                Agent                          Server
                                                            Metric
                  │                                      │  Backend
                  │ServerToAgent{ConnectionSettingsOffer}│
                  │◄─────────────────────────────────────┤    │
                  │                                      │    │
                  │                                           │
    ┌────────┐    │                                           │
    │Collect │    │                OTLP Metrics               │ ──┐
    │Own     ├───►├──────────────────────────────────────────►│   │
    │Metrics │    │                                           │   │
    └────────┘    .                    ...                    .   │ Repeats
                                                                  │
    ┌────────┐    │                                           │   │ Periodically
    │Collect │    │                OTLP Metrics               │   │
    │Own     ├───►├──────────────────────────────────────────►│   │
    │Metrics │    │                                           │ ──┘
    └────────┘    │                                           │
```


The Agent SHOULD report metrics of the agent process (or processes) and any
custom metrics that describe the agent state. Reported process metrics MUST
follow the OpenTelemetry
[conventions for processes](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/metrics/semantic_conventions/process-metrics.md).

Similarly, the Agent SHOULD report its traces to the destination offered in the
[own_traces](#own_traces) field and logs to the destination offered in the
[own_logs](#own_logs) field.

All attributes specified in the
[identifying_attributes](#identifying_attributes) field in AgentDescription
message SHOULD be also specified in the Resource of the reported OTLP telemetry.

Attributes specified in the
[non_identifying_attributes](#non_identifying_attributes) field in
AgentDescription message may be also specified in the Resource of the reported
OTLP telemetry, in which case they SHOULD have exactly the same values.


<h2 id="configuration">Configuration</h2>


Agent configuration is an optional capability of OpAMP protocol. Remote
configuration capability can be disabled if necessary (for example when using
existing configuration capabilities of an orchestration system such as
Kubernetes).

The Server can offer a Remote Configuration to the Agent by setting the
[remote_config](#remote_config) field in the ServerToAgent message. Since the
ServerToAgent message is normally sent by the Server in response to a status
report the Server has the Agent's description and may tailor the configuration
it offers to the specific Agent if necessary.

The Agent MUST set the AcceptsRemoteConfig bit of StatusReport.capabilities if
the Agent is capable of accepting remote configuration. If the bit is not set
the Server MUST not offer a remote configuration to the Agent.

The Agent's actual configuration that it uses for running may be different from
the Remote Configuration that is offered by the Server. This actual
configuration is called the Effective Configuration of the Agent. The Effective
Configuration is typically formed by the Agent after merging the Remote
Configuration with other inputs available to the Agent, e.g. a locally available
configuration.

Once the Effective Configuration is formed the Agent uses it for its operation
and will also report the Effective Configuration to the OpAMP Server via the
[effective_config](#effective_config) field of status report. The Server
typically allows the end user to see the effective configuration alongside other
data reported in the status reported by the Agent.

The Agent MUST set the ReportsEffectiveConfig bit of StatusReport.capabilities
if the Agent is capable of reporting effective configuration. If the bit is not
set the Server should not expect the StatusReport.effective_config field to be
set.

Here is the typical configuration sequence diagram:


```
               Agent                              Server

                 │ AgentToServer{StatusReport}       │   ┌─────────┐
                 ├──────────────────────────────────►├──►│ Process │
                 │                                   │   │ Status  │
Local     Remote │                                   │   │ and     │
Config    Config │ ServerToAgent{AgentRemoteConfig}  │   │ Fetch   │
  │     ┌────────┤◄──────────────────────────────────┤◄──┤ Config  │
  ▼     ▼        │                                   │   └─────────┘
┌─────────┐      │                                   │
│ Config  │      │                                   │
│ Merger  │      │                                   │
└────┬────┘      │                                   │
     │           │                                   │
     │Effective  │                                   │
     │Config     │ AgentToServer{StatusReport}       │
     └──────────►├──────────────────────────────────►│
                 │                                   │
                 │                                   │
```


Note: the Agent SHOULD NOT send a status report if the Effective Configuration
or other fields that are reported via StatusReport message are unchanged. If the
Agent does not follow this rule the operation may result in an infinite loop of
messages sent back and forth between the Agent and the Server.

The Server may also initiate sending of a remote configuration on its own,
without waiting for a status report from the Agent. This can be used to
re-configure an Agent that is connected but which has nothing new to report. The
sequence diagram in this case looks like this:


```
               Agent                              Server

                 │                                   │
                 │                                   │
                 │                                   │   ┌────────┐
Local     Remote │                                   │   │Initiate│
Config    Config │  ServerToAgent{AgentRemoteConfig} │   │and     │
  │     ┌────────┤◄──────────────────────────────────┤◄──┤Send    │
  ▼     ▼        │                                   │   │Config  │
┌─────────┐      │                                   │   └────────┘
│ Config  │      │                                   │
│ Merger  │      │                                   │
└────┬────┘      │                                   │
     │           │                                   │
     │Effective  │                                   │
     │Config     │ AgentToServer{StatusReport}       │
     └──────────►├──────────────────────────────────►│
                 │                                   │
                 │                                   │
```


The Agent may ignore the Remote Configuration offer if it does not want its
configuration to be remotely controlled by the Server.

<h3 id="configuration-files">Configuration Files</h3>


The configuration of the Agent is a collection of named configuration files
(this applies both to the Remote Configuration and to the Effective
Configuration).

The file names MUST be unique within the collection. It is possible that the
Remote and Local Configuration MAY contain a file with the same name but with a
different content. How these files are merged to form an Effective Configuration
is agent type-specific and is not part of the OpAMP protocol.

If there is only one configuration file in the collection then the file name MAY
be empty.

The collection of configuration files is represented using a AgentConfigMap
message:


```protobuf
message AgentConfigMap {
  map<string, AgentConfigFile> config_map = 1;
}
```


The config_map field of the AgentConfigSet message is a map of configuration
files, where keys are file names.

For agents that use a single config file the config_map field SHOULD contain a
single entry and the key MAY be an empty string.

The AgentConfigFile message represents one configuration file and has the
following structure:


```protobuf
message AgentConfigFile {
  bytes body = 1;
  string content_type = 2;
}
```


The body field contains the raw bytes of the configuration file. The content,
format and encoding of the raw bytes is agent type-specific and is outside the
concerns of OpAMP protocol.

content_type is an optional field. It is a MIME Content-Type that describes
what's contained in the body field, for example "text/yaml". The content_type
reported in the Effective Configuration in the Agent's status report may be used
for example by the Server to visualize the reported configuration nicely in a
UI.

<h3 id="security-considerations">Security Considerations</h3>


Remote Configuration is a potentially dangerous functionality that may be
exploited by malicious actors. For example if the Agent is capable of collecting
local files and sending over the network then a compromised OpAMP Server may
offer a malicious remote configuration to the Agent and compel the Agent to
collect a sensitive local file and send to a specific network destination.

See Security section for [general recommendations](#general-recommendations) and
recommendations specifically for
[remote reconfiguration](#configuration-restrictions) capabilities.

<h3 id="agentremoteconfig-message">AgentRemoteConfig Message</h3>


The message has the following structure:


```protobuf
message AgentRemoteConfig {
  AgentConfigMap config = 1;
}
```


<h2 id="addons">Addons</h2>


An Addon has a name and content stored in a file. The content of the file,
functionality provided by the addons, how they are stored and used by the Agent
side is agent type-specific and is outside the concerns of the OpAMP protocol.

The Agent may have zero or more addons installed. Each addon has a name. The
Agent cannot have more than one addon of the particular name installed.

Different addons may have files with the same name, file names are not globally
unique, they are only unique within the scope of a particular addon.

Addons may be provided and installed locally (e.g. by a local user). The addons
may be also offered to the Agent remotely by the Server, in which case the Agent
may download and install the addons.

To offer addons to the Agent the Server sets the
[addons_available](#servertoagent-message) field in the ServerToAgent message that
is sent either in response to an status report form the Agent or by Server's
initiative if the Server wants to push addons to the Agent.

The [AddonsAvailable](#addonsavailable-message) message describes the addons
that are available on the Server for this Agent. For each addon the message
describes the file that has the addon content and provides the URL from which
the file can be downloaded using an HTTP GET request. The URLs point to addon
files on a Download Server (which may be on the same host as the OpAMP Server or
a different host).

The protocol supports only a single downloadable file per addon. If the Agent's
addons conceptually are composed of multiple files then the Agent and Server can
agree to store the files in any file format that allows storing multiple files
in a single file, e.g. a zip or tar file. After downloading the single addon
file the Agent may extract the files contained in it. How exactly this is done
is Agent specific and is beyond the scope of the protocol.

The Server is allowed to make an addon offer only if the Agent indicated that it
can accept addons via AcceptsAddons bit of StatusReport.capabilities field.

<h3 id="downloading-addons">Downloading Addons</h3>


After receiving the [AddonsAvailable](#addonsavailable-message) message the
Agent SHOULD follow this download procedure:

<h4 id="step-1">Step 1</h4>


Compare the aggregate hash of all addons it has with the aggregate hash offered
by the Server in the all_addons_hash field.

If the aggregate hash is the same then consider the download procedure done,
since it means all addons on the Agent are the same as offered by the Server.
Otherwise go to Step 2.

<h4 id="step-2">Step 2</h4>


For each addon offered by the Server the Agent SHOULD check if it should
download the particular addon:



* If the Agent does not have an addon with the specified name then it SHOULD
  download the addon. See Step 3 on how to download each addon file.
* If the Agent has the addon the Agent SHOULD compare the hash of the addon that
  the Agent has with the hash of the addon offered by the Server in the
  [hash](#hash) field in the [AddonAvailable](#addonavailable-message) message.
  If the hashes are the same then the addon is the same and processing for this
  addon is done, proceed to the next addon. If the hashes are different then
  check the addon file as described in Step 3.

Finally, if the Agent has any addons that are not offered by the Server the
addons SHOULD be deleted by the Agent.

<h4 id="step-3">Step 3</h4>

For the file of the addon offered by the Server the Agent SHOULD check if it
should download the file:

* If the Agent does not have a file with the specified name then it SHOULD
  download the file.
* If the Agent has the file then the Agent SHOULD compare the hash of the file
  it has locally with the [hash](#hash) field in the
  [DownloadableFile](#downloadablefile-message) message. If hashes are the same
  the processing of this file is done. Otherwise the offered file is different
  and the file SHOULD be downloaded from the location specified in the
  [download_url](#download_url) field of the
  [DownloadableFile](#downloadablefile-message) message. The Agent SHOULD use an
  HTTP GET message to download the file.

The procedure outlined above allows the Agent to efficiently download only new
or changed addons and only download new or changed files.

After downloading the addons the Agent can perform any additional processing
that is agent type-specific (e.g. "install" or "activate" the addons in any way
that is specific to the agent).

<h3 id="addon-status-reporting">Addon Status Reporting</h3>


During the downloading and installation process the Agent MAY periodically
report the status of the process. To do this the Agent SHOULD send an
[AgentToServer](#agenttoserver-message) message and set the [Body](#body) to
[AgentAddonStatuses](#agentaddonstatuses-message) message.

Once the downloading and installation of all addons is done (succeeded or
failed) the Agent SHOULD report the status of all addons to the Server.

Here is the typical sequence diagram for the addon downloading and status
reporting process:


```
    Download           Agent                             OpAMP
     Server                                              Server
       │                 │                                  │
       │                 │  ServerToAgent{AddonsAvailable}  │
       │                 │◄─────────────────────────────────┤
       │   HTTP GET      │                                  │
       │◄────────────────┤                                  │
       │ Download file #1│                                  │
       ├────────────────►│                                  │
       │                 │ AgentToServer{AgentAddonStatuses}│
       │                 ├─────────────────────────────────►│
       │   HTTP GET      │                                  │
       │◄────────────────┤                                  │
       │ Download file #2│                                  │
       ├────────────────►│                                  │
       │                 │ AgentToServer{StatusReport}      │
       │                 ├─────────────────────────────────►│
       │                 │                                  │
       .                 .                                  .

       │   HTTP GET      │                                  │
       │◄────────────────┤                                  │
       │ Download file #N│                                  │
       ├────────────────►│                                  │
       │                 │ AgentToServer{AgentAddonStatuses}│
       │                 ├─────────────────────────────────►│
       │                 │                                  │
```


The Agent MUST always include all addons it has or is processing (downloading or
installing) in AgentAddonStatuses message.

Note that the Agent MAY also report the status of addons it has installed
locally, not only the addons it was offered and downloaded from the Server.
[TODO: is this necessary?]

<h3 id="calculating-hashes">Calculating Hashes</h3>


The Agent and the Server use hashes to identify content of files and addons such
that the Agent can decide what files and addons need to be downloaded.

The calculation of the hashes is performed by the Server. The server MUST choose
a strong hash calculation method with minimal collision probability (and it may
seed random values into calculation to guarantee hash uniqueness if such
guarantees are needed by the implementation).

The hashes are opaque to the Agent, the Agent never calculates hashes, it only
stores and compares them.

There are 3 levels of hashes: 

<h4 id="file-hash">File Hash</h4>

The hash of the addon file content. This is stored in the [content_hash](#content_hash) field in
the [DownloadableFile](#downloadablefile-message) message. This value SHOULD be
used by the Agent to determine if the particular file it has is different on the
Server and needs to be re-downloaded.

<h4 id="addon-hash">Addon Hash</h4>

The addon hash that identifies the entire addon (addon name and file content).
This hash is stored in the [hash](#hash) field in the
[AddonAvailable](#addonavailable-message) message.

This value SHOULD be used by the Agent to determine if the particular addon it
has is different on the Server and needs to be re-downloaded.

<h4 id="all-addons-hash">All Addons Hash</h4>


The all addons hash is the aggregate hash of all addons for the particular
Agent. The hash is calculated as an aggregate of all addon names and content.
This hash is stored in the [all_addons_hash](#all_addons_hash) field in the
[AddonsAvailable](#addonsavailable-message) message.

This value SHOULD be used by the Agent to determine if any of the addons it has
are different from the ones available on the Server and need to be
re-downloaded.

Note that the aggregate hash does not include the addons that are available on
the Agent locally and were not downloaded from the download server.

<h3 id="security-considerations">Security Considerations</h3>


Downloading addons remotely is a potentially dangerous functionality that may be
exploited by malicious actors. If addons contain executable code then a
compromised OpAMP Server may offer a malicious addon to the Agent and compel the
Agent to execute arbitrary code.

See Security section for [general recommendations](#general-recommendations) and
recommendations specifically for [code signing](#code-signing) capabilities.

<h3 id="addonsavailable-message">AddonsAvailable Message</h3>


The message has the following structure:


```
message AddonsAvailable {
    map<string, AddonAvailable> addons = 1;
    bytes all_addons_hash = 2;
}
```


<h4 id="addons">addons</h4>


A map of addons. Keys are addon names.

<h4 id="all_addons_hash">all_addons_hash</h4>


Aggregate hash of all remotely installed addons.

The agent SHOULD include this value in subsequent
[StatusReport](#statusreport-message) messages. This in turn allows the Server
to identify that a different set of addons is available for the agent and
specify the available addons in the next DataToAgent message.

This field MUST be always set if the Server supports addons of agents.

<h3 id="addonavailable-message">AddonAvailable Message</h3>


This message is an offer from the Server to the agent to install a new addon or
initiate an upgrade or downgrade of an addon that the Agent already has. The
message has the following structure:


```protobuf
message AddonAvailable {
    string version = 1;
    DownloadableFile file = 2;
    bytes hash = 3;
}
```


TODO: do we need other fields, e.g. description?

#### version

The addon version that is available on the server side. The agent may for
example use this information to avoid downloading an addon that was previously
already downloaded and failed to install.

#### file

The downloadable file of the addon.

#### hash

The hash of the addon. SHOULD be calculated based on addon name and
content of the file of the addon.

<h3 id="downloadablefile-message">DownloadableFile Message</h3>

The message has the following structure:

```protobuf
message DownloadableFile {
    string download_url = 1;
    bytes content_hash = 2;
    bytes signature = 3;
}
```

<h4 id="download_url">download_url</h4>

The URL from which the file can be downloaded using HTTP GET request. The server
at the specified URL SHOULD support range requests to allow for resuming
downloads.

#### content_hash

The SHA256 hash of the file content. Can be used by the Agent to verify that the file
was downloaded correctly.

#### signature

Optional signature of the file content. Can be used by the Agent to verify the
authenticity of the downloaded file, for example can be the 
[detached GPG signature](https://www.gnupg.org/gph/en/manual/x135.html#AEN160).
The exact signing and verification method is Agent specific. See 
[Code Signing](#code-signing) for recommendations.

<h2 id="agent-package-updates">Agent Package Updates</h2>

Agent package is a downloadable file. The package can be downloaded by the Agent
and installed to replace the Agent itself, either to upgrade it to a newer
version or to downgrade it to an older version. The content of the file, how it
is installed on the Agent side is agent type-specific and is outside the
concerns of the OpAMP protocol.

To offer a package to the Agent the Server sets the
[agent_package_available](#agent_package_available) field in the ServerToAgent
message that is sent either in response to an status report form the Agent or by
Server's initiative if the Server wants to push a package to the Agent.

The [AgentPackageAvailable](#agentpackageavailable-message) message describes a
package that is available on the Server for this Agent. The message points to
the file that is contains the package and provides the URL from which the file
can be downloaded using an HTTP GET request. The URLs point to a file on a
Download Server (which may be on the same host as the OpAMP Server or a
different host).

The Server is allowed to make an package offer only if the Agent indicated that
it can accept packages via AcceptsAgentPackage bit of StatusReport.capabilities
field.

<h3 id="downloading-agent-package">Downloading Agent Package</h3>

After receiving the [AgentPackageAvailable](#agentpackageavailable-message)
message the Agent SHOULD follow the download procedure that is similar to
[Addon download procedure](#downloading-addons) and download the package file
as necessary, using the hashes to avoid unnecessary downloads.

The Agent MAY send status reports as it downloads the file. The Agent SHOULD
send a status report when the downloading of the package is finished and the
package is installed (or failed to install). The execution sequence is the
following:


```
    Download           Agent                              OpAMP
     Server                                               Server
       │                 │                                    │
       │                 │ServerToAgent{AgentPackageAvailable}│
       │                 │◄───────────────────────────────────┤
       │   HTTP GET      │                                    │
       │◄────────────────┤                                    │
       │ Download file   │                                    │
       ├────────────────►│                                    │
       │                 │ AgentToServer{StatusReport}        │
       │                 ├───────────────────────────────────►│
       │                 │                                    │
```


It is recommended that before applying the downloaded package the Agent saves
the current state and is ready to rollback the installation if it fails, such
that the Agent does not end up in a broken state due to a failed attempt to
install the downloaded package. The exact mechanisms for such rollback are
outside the scope of the OpAMP specification.

The protocol supports only a single downloadable file per package. If the
Agent's packages conceptually are composed of multiple files then the Agent and
Server can agree to store the files in any file format that allows storing
multiple files in a single file, e.g. a zip or tar file. After downloading the
single package file the Agent may extract the files contained in it. How exactly
this is done is Agent specific and is beyond the scope of the protocol.

<h3 id="security-considerations">Security Considerations</h3>


Downloading executable agent packages remotely is a potentially dangerous
functionality that may be exploited by malicious actors. A compromised OpAMP
Server may offer a malicious executable to the Agent and compel the Agent to
execute arbitrary code.

See Security section for [general recommendations](#general-recommendations) and
recommendations specifically for [code signing](#code-signing) capabilities.

<h3 id="agentpackageavailable-message">AgentPackageAvailable Message</h3>


The message is sent from the server to the agent to indicate that there is an
agent package available for the agent to download and self-update. The message
has the following structure:


```protobuf
message AgentPackageAvailable {
    string version = 1;
    DownloadableFile file = 2;
}
```


<h4 id="version">version</h4>

The agent version that is available on the server side. The agent may for
example use this information to avoid downloading a package that was previously
already downloaded and failed to install.

<h4 id="file">file</h4>

The downloadable file of the package.

<h1 id="connection-management">Connection Management</h1>


## Establishing Connection

The Agent connects to the Server by establishing an HTTP(S) connection, then
upgrading the connection to WebSocket as defined by WebSocket standard. After
the WebSocket connection is established the Agent MUST send the first
[status report](#status-reporting) and expect a response to it.

If the Agent is unable to establish a WebSocket connection to the Server it
SHOULD retry connection attempts and use exponential backoff strategy with
jitter to avoid overwhelming the Server.

When retrying connection attempts the Agent SHOULD honour any
[throttling](#throttling) responses it receives from the Server.

<h2 id="closing-connection">Closing Connection</h2>


<h3 id="agent-initiated">Agent Initiated</h3>


To close a connection the Agent MUST first send an AgentToServer message with
agent_disconnect field set. The Agent MUST then send a WebSocket
[Close](https://datatracker.ietf.org/doc/html/rfc6455#section-5.5.1) control
frame and follow the procedure defined by WebSocket standard.

<h3 id="server-initiated">Server Initiated</h3>


To close a connection the Server MUST then send a WebSocket
[Close](https://datatracker.ietf.org/doc/html/rfc6455#section-5.5.1) control
frame and follow the procedure defined by WebSocket standard.

## Restoring Connection

If an established WebSocket connection is broken (disconnected) unexpectedly the
Agent SHOULD immediately try to re-connect. If the re-connection fails the Agent
SHOULD continue connection attempts with backoff as described in
[Establishing Connection](#establishing-connection).


<h2 id="duplicate-connections">Duplicate Connections</h2>


Each Agent instance SHOULD connect no more than once to the Server. If the Agent
needs to re-connect to the Server the Agent MUST ensure that it sends an
AgentDisconnect message first, then closes the existing connection and only then
attempts to connect again.

The Server MAY disconnect or deny serving requests if it detects that the same
Agent instance has more than one simultaneous connection or if multiple Agent
instances are using the same instance_uid.

The Server SHOULD detect duplicate instance_uids (which may happen for example
when Agents are using bad UID generators or due to cloning of the VMs where the
Agent runs). When a duplicate instance_uid is detected, Server SHOULD generate
a new instance_uid, and send it as new_instance_uid value of AgentIdentification.

<h2 id="authentication">Authentication</h2>


The Agent and the Server MAY use authentication methods supported by HTTP, such
as [Basic](https://datatracker.ietf.org/doc/html/rfc7617) authentication or
[Bearer](https://datatracker.ietf.org/doc/html/rfc6750) authentication. The
authentication happens when the HTTP connection is established before it is
upgraded to a WebSocket connection.

The Server MUST respond with
[401 Unauthorized](https://datatracker.ietf.org/doc/html/rfc7235#section-3.1) if
the Agent authentication fails.

<h2 id="bad-request">Bad Request</h2>


If the Server receives a malformed AgentToServer message the Server SHOULD
respond with a ServerToAgent message with error_response field set in the
[Body](#body) and the [Type](#type) of
[ServerErrorResponse](#errorresponse-message) message set to BAD_REQUEST. The
[error_message](#error_message) field SHOULD be a human readable description of
the problem with the AgentToServer message.

The Agent SHOULD NOT retry sending an AgentToServer message to which it received
a BAD_REQUEST response.

<h2 id="retrying-messages">Retrying Messages</h2>


The Agent MAY retry sending AgentToServer message if:


* AgentToServer message that requires a response was sent, however no response
  was received within a reasonable time (the timeout MAY be configurable).
* AgentToServer message that requires a response was sent, however the
  connection was lost before the response was received.
* After receiving an UNAVAILABLE response from the Server as described in the
  [Throttling](#throttling) section.

For messages that require a response if the Server receives the same message
more than once the Server MUST respond to each message, not just the first
message, even if the Server detects the duplicates and processes the message
once.

Note that the Agent is not required to keep a growing queue of messages that it
wants to send to the Server if the connection is unavailable. The Agent
typically only needs to keep one up-to-date message of each kind that it wants
to send to the Server and send it as soon as the connection is available.

For example, the Agent should keep track of its own status and compose a
StatusReport message that is ready to be sent at the first opportunity. If the
Agent is unable to send the StatusReport message (for example if the connection
is not yet available) the Agent does not need to create a new StatusReport every
time the Agent's status changes and keep all these StatusReport messages in a
queue ready to be sent. The Agent simply needs to keep one up-to-date
StatusReport message and send it at the first opportunity. This of course
requires the StatusReport message to contain all changes since it was last
reported and to correctly reflect the current (last) state of the Agent.

Similarly, all other Agent reporting capabilities, such as Addon Status
Reporting or Agent Package Installation Status Reporting require the Agent to
only keep one up-to-date status message and send it at the earliest opportunity.

The exact same logic is true in the opposite direction: the Server normally only
needs to keep one up-to-date message of a particular kind that it wants to
deliver to the Agent and send it as soon as the connection to the Agent is
available.


## Throttling

When the Server is overloaded and is unstable to process the AgentToServer
message it SHOULD respond with an ServerToAgent message with error_response
field set in the [Body](#body) and the [type](#type) of
[ServerErrorResponse](#servererrorresponse-message) message set to UNAVAILABLE.
~~The agent SHOULD retry the message.~~ _(Note: retrying individual messages is
not possible since we no longer have sequence ids and don't know which message
failed)._ The agent SHOULD disconnect, wait, then reconnect again and resume its
operation. The retry_info field may be optionally set with
retry_after_nanoseconds field specifying how long the Agent SHOULD wait before
~~retiring the message~~ reconnecting:


```protobuf
message RetryInfo {
    uint64 retry_after_nanoseconds = 1;
}
```


If retry_info is not set then the Agent SHOULD implement an exponential backoff
strategy to gradually increase the interval between retries.

When the Server is overloaded it may also be unable to upgrade the HTTP
connection to WebSocket. The Server MAY return
[HTTP 503 Service Unavailable](https://datatracker.ietf.org/doc/html/rfc7231#page-63)
or [HTTP 429 Too Many Requests](https://datatracker.ietf.org/doc/html/rfc6585#section-4)
response and MAY optionally set
[Retry-After](https://datatracker.ietf.org/doc/html/rfc7231#section-7.1.3)
header to indicate when SHOULD the Agent attempt to reconnect. The Agent SHOULD
honour the corresponding requirements of HTTP specification.

The minimum recommended retry interval is 30 seconds.

<h1 id="security">Security</h1>


Remote configuration, downloadable addons and agent packages are a significant
security risk. By sending a malicious server-side configuration or a malicious
addon the Server may compel the Agent to perform undesirable work. This section
defines recommendations that reduce the security risks for the Agent.

Guidelines in this section are optional for implementation, but are highly
recommended for sensitive applications.

<h2 id="general-recommendations">General Recommendations</h2>


We recommend that the Agent employs the zero-trust security model and does not
automatically trust the remote configuration or other offers it receives from
the Server. The data received from the Server SHOULD be verified and sanitized
by the Agent in order to limit and prevent the damage that may be caused by
malicious actors. We recommend the following:



* The Agent SHOULD run at the minimum possible privilege to prevent itself from
  accessing sensitive files or perform high privilege operations. The Agent
  SHOULD NOT run as root user, otherwise a compromised Agent may result in total
  control of the machine by malicious actors.
* If the Agent is capable of collecting local data it SHOULD limit the
  collection to a specific set of directories. This limitation SHOULD be locally
  specified and SHOULD NOT be overridable via remote configuration. If this rule
  is not followed the remote configuration functionality may be exploited to
  access sensitive information on the Agent's machine.
* If the Agent is capable of executing external code located on the machine
  where it runs and this functionality can be specified in the Agent's
  configuration then the Agent SHOULD limit such functionality only to specific
  scripts located in a limited set of directories. This limitation SHOULD be
  locally specified and SHOULD NOT be overridable via remote configuration. If
  this rule is not followed the remote configuration functionality may be
  exploited to perform arbitrary code execution on the Agent's machine.

<h2 id="configuration-restrictions">Configuration Restrictions</h2>


The Agent is recommended to restrict what it may be compelled to do via remote
configuration.

Particularly, if it is possible via a configuration to ask the Agent to collect
data from the machine it runs on (as it is often the case for telemetry
collecting agents) then we recommend to have agent-side restrictions as to what
directories or files the Agent is allowed to collect. Upon receiving a remote
config the Agent must validate the configuration against the list of
restrictions and refuse to apply the configuration either fully or partially if
it violates the restrictions or sanitize the configuration such that it does not
collect data from prohibited directories or files.

Similarly, if the configuration provides means to order the Agent to execute
processes or scripts on the machine it runs on we recommend to have agent-side
restrictions as to what executable files from what directories the Agent is
allowed to run.

It is recommended that the restrictions are specified in the form of "allow
list" instead of the "deny list". The restrictions may be hard-coded or may be
end-user definable in a local config file. It should not be possible to override
these restrictions by sending a remote config from the Server to the agent.

<h2 id="opt-in-remote-configuration">Opt-in Remote Configuration</h2>


It is recommended that remote configuration capabilities are not enabled in the
Agent by default. The capabilities should be opt-in by the user.

## Code Signing


Any executable code that is part of an addon or agent package should be signed
to prevent a compromised Server from delivering malicious code to the Agent. We
recommend the following:



* Any downloadable executable code (e.g. executable addons or agent packages)
  need to be code-signed. The actual code-signing and verification mechanism is
  agent specific and is outside the concerns of the OpAMP specification.
* The Agent SHOULD verify executable code in downloaded files to ensure the code
  signature is valid.
* The downloadable code can be signed with the signature included in the file content or 
  have a detached signature recorded in the DownloadableFile
  message's [signature](#signature) field. Detached signatures may be used for example
  with [GPG signing](https://www.gnupg.org/gph/en/manual/x135.html#AEN160).
* If Certificate Authority is used for code signing it is recommended that the
  Certificate Authority and its private key is not co-located with the OpAMP
  Server, so that a compromised Server cannot sign malicious code.
* The Agent SHOULD run any downloaded executable code (the addons and or any
  code that it runs as external processes) at the minimum possible privilege to
  prevent the code from accessing sensitive files or perform high privilege
  operations. The Agent SHOULD NOT run downloaded code as root user.

# Interoperability

## Interoperability of Partial Implementations

OpAMP defines a number of capabilities for the Agent and the Server. Most of
these capabilities are optional. The Agent or the Server should be prepared that
the peer does not support a particular capability.

Both the Agent and the Server indicate the capabilities that they support during
the initial message exchange. The Agent sets the capabilities bit-field in the
StatusReport message, the Server sets the capabilities bit-field in the
ServerToAgent message.

Each set bit in the capabilities field indicates that the particular capability
is supported. The list of Agent capabilities is [here](#statusreport-message).
The list of Server capabilities is [here](#servertoagent-message).

After the Server learns about the capabilities of the particular Agent the
Server MUST stop using the capabilities that the Agent does not support.

Similarly, after the Agent learns about the capabilities of the Server the Agent
MUST stop using the capabilities that the Server does not support.

The specifics of what in the behavior of the Agent and the Server needs to
change when they detect that the peer does not support a particular capability
are described in this document where relevant.

## Interoperability of Future Capabilities

There are 2 ways OpAMP enables interoperability between an implementation of the
current version of this specification and an implementation of a future,
extended version of OpAMP that adds more capabilities that are not described in
this specification.

### Ignorable Capability Extensions

For the new capabilities that extend the functionality in such a manner that
they can be silently ignored by the peer a new field may be added to any
Protobuf message. The sender that implements this new capability will set the
new field. A recipient that implements an older version of the specification
that is unaware of the new capability will simply ignore the new field. The
Protobuf encoding ensures that the rest of the fields are still successfully
deserialized by the recipient.

### Non-Ignorable Capability Extensions 

For the new capabilities that extend the functionality in such a manner that
they cannot be silently ignored by the peer a different approach is used.

The capabilities fields in StatusReport and ServerToAgent messages contains a
number of reserved bits. These bits SHOULD be used for indicating support of new
capabilities that will be added to OpAMP in the future.

The Agent and the Server MUST set these reserved bits to 0 when sending the
message. This allows the recipient, which implements a newer version of OpAMP to
learn that the sender does not support the new capability and adjust its
behavior correspondingly.

The StatusReport and ServerToAgent messages are the first messages exchanged by
the Agent and Server which allows them to learn about the capabilities of the
peer and adjust their behavior appropriately. How exactly the behavior is
adjusted for future capabilities MUST be defined in the future specification of
the new capabilities.

<h1 id="performance-and-scale">Performance and Scale</h1>


TBD

<h1 id="open-questions">Open Questions</h1>




* How does the server know that the request/report received actually is
  generated by the agent that the message claims? Do we need to verify that the
  message is not fake (impersonated) or the server trusts the senders that are
  authenticated?
* ~~In Otel Helm chart how are Agents on k8s nodes and the standalone Collector
  are managed? Do we consider the entire Otel Helm chart to be a single Agent or
  each individual instance on each node plus the standalone Collector are
  separate Agents from management perspective?~~ Discussed with Dmitry and
  concluded that each need to be separate agent instances.
* Do we need to define OpenTelemetry semantic conventions for reporting typical
  collection Agent-specific metrics (e.g. input/processing/output data rates,
  throughput, latency, etc)?
* ~~Do we need a capability for the Server to order the Agent to restart?~~ Added.
* Do we need Agent-initiated client certificate rotation capability (in addition
  to Server-initiated that we already have)?
* Do we need to recommend the Agent to cache the remote config and OTLP metric
  destination locally in case the Server is unavailable to make the system more
  resilient?
* ~~Do we need to make initial (first time after connection) configuration
  fetching more efficient if it is unchanged since it was fetched during the
  previous connection session (by exchanging config hashes first)? May be
  important if remote configuration can be very large and/or Agents reconnect
  frequently. If using hashes should this be one aggregate hash per config file
  collection or individual hashes per config file?~~ Done.
* ~~Do we need the sequence_num concept?~~ Deleted for now, not necessary for
  current feature set, but may need to be restored for other features (e.g.
  custom "extensions").
* ~~Does the Server need to actively detect duplicate instance_uids, which may
  happen due to Agents using bad UID generators which create globally non-unique
  UIDs?~~ Added.
* ~~Do we need to split the AddonStatus and AgentStatus from the general
  StatusReport?~~ Yes, splitted.
* Does WebSocket frame compression work for us or do we need our own mechanism?
* What are server CPU/RAM, network resources requirements and intermediary
  capabilities requirements at scale (millions of agents)? Test and fill the
  blanks in the [Performance and Scale](#performance-and-scale) section.
* How does TLS work with cloud provider load balancers? Do we terminate TLS at
  load balancer and if yes how does the server know the client's certificate for
  acceptance or to rotate? Add section on Load Balancing to explain cons and
  pros of TLS-terminating vs non-terminating load balancers (client certificate
  invisible to the server when TLS-terminating).
* ~~Do we need to add log and trace destinations?~~ Added.
* ~~Do we need access token rotation? Use the initial token and generate on the
  first connection.~~ Added.
* ~~Can we unite certificate offer and access token offer?~~ Unified.
* ~~Can we have multiple offers of OTHER type?~~ Now using named "other"
  connections.
* ~~Do we need connection status reporting or it can be deleted?~~ Deleted.
* Do we need to allow for "extensions" to the protocol so that custom messages
  may be exchanged between the Agent and the Server in the same WebSocket
  connection?

<h1 id="faq-for-reviewers">FAQ for Reviewers</h1>


<h3 id="what-is-websocket">What is WebSocket?</h3>


WebSocket is a bidirectional, message-oriented protocol that uses plain HTTP for
establishing the connection and then uses the HTTP's existing TCP connection to
deliver messages. It has been an
[RFC](https://datatracker.ietf.org/doc/html/rfc6455) standard for a decade now.
It is widely supported by browsers, servers, proxies and load balancers, has
libraries in virtually all popular programming languages, is supported by
network inspection and debugging tools, is secure and efficient and provides the
exact message-oriented semantics that we need for OpAMP.

<h3 id="why-not-use-tcp-instead-of-websocket">Why not Use TCP Instead of WebSocket?</h3>


We could roll out our own message-oriented implementation over TCP but there are
no benefits over WebSocket which is an existing widely supported standard. A
custom TCP-based solution would be more work to design, more work to implement
and more work to troubleshoot since existing network tools would not recognize
it.

<h3 id="why-not-use-http-instead-of-websocket">Why not Use HTTP Instead of WebSocket?</h3>


Regular HTTP is a half-duplex protocol, which makes delivery of messages from
the server to the client tied to the request time of the client. This means that
if the server needs to send a message to the client the client either needs to
periodically poll the server to give the server an opportunity to send a message
or we should use something like long polling.

Periodic polling is expensive. OpAMP protocol is largely idle after the initial
connection since there is typically no data to deliver for hours or days. To
have a reasonable delivery latency the client would need to poll every few
seconds and that would significantly increase the costs on the server side (we
aim to support many millions simultaneous of Agents, which would mean servicing
millions of polling requests per second).

Long polling is more complicated to use than WebSocket since it only provides
one-way communication, from the server to the client and necessitates the second
connection for client-to-server delivery direction. The dual connection needed
for a long polling approach would make the protocol more complicated to design
and implement without much gains compared to WebSocket approach.

<h3 id="why-not-use-grpc-instead-of-websocket">Why not Use gRPC Instead of WebSocket?</h3>


gRPC is a big dependency that some implementations are reluctant to take. gRPC
requires HTTP/2 support from all intermediaries and is not supported in some
load balancers. As opposed to that, WebSocket is usually a small library in most
language implementations (or is even built into runtime, like it is in browsers)
and is more widely supported by load balancers since it is based on HTTP/1.1
transport.

Feature-wise gRPC streaming would provide essentially the same functionality as
WebSocket messages, but it is a more complicated dependency that has extra
requirements with no additional benefits for our use case (benefits of gRPC like
ability to multiplex multiple streams over one connection are of no use to
OpAMP).

<h1 id="future-possibilities">Future Possibilities</h1>


Define specification for Concentrating Proxy that can serve as intermediary to
reduce the number of connections to the Server when a very large number
(millions and more) of Agents are managed.

OpAMP may be extended by a polling-based HTTP standard. It will have somewhat
worse latency characteristics but may be desirable for some implementation.

<h1 id="references">References</h1>


<h2 id="agent-management">Agent Management</h2>




* Splunk
  [Deployment Server](https://docs.splunk.com/Documentation/Splunk/8.2.2/Updating/Aboutdeploymentserver).
* Centralized Configuration of vRealize
  [Log Insight Agents](https://docs.vmware.com/en/vRealize-Log-Insight/8.4/com.vmware.log-insight.agent.admin.doc/GUID-40C13E10-1554-4F1B-B832-69CEBF85E7A0.html).
* Google Cloud
  [Guest Agent](https://github.com/GoogleCloudPlatform/guest-agent) uses HTTP
  [long polling](https://cloud.google.com/compute/docs/metadata/querying-metadata#waitforchange).

<h2 id="configuration-management">Configuration Management</h2>




* [Uber Flipr](https://eng.uber.com/flipr/).
* Facebook's
  [Holistic Configuration Management](https://research.fb.com/wp-content/uploads/2016/11/holistic-configuration-management-at-facebook.pdfhttps://research.fb.com/wp-content/uploads/2016/11/holistic-configuration-management-at-facebook.pdf)
  (push).

<h2 id="security-and-certificate-management">Security and Certificate Management</h2>




* mTLS in Go:
  [https://kofo.dev/how-to-mtls-in-golang](https://kofo.dev/how-to-mtls-in-golang)
* e2e audit
  [https://pwn.recipes/posts/roll-your-own-e2ee-protocol/](https://pwn.recipes/posts/roll-your-own-e2ee-protocol/)
* ACME certificate management protocol
  [https://datatracker.ietf.org/doc/html/rfc8555](https://datatracker.ietf.org/doc/html/rfc8555)
* ACME for client certificates
  [http://www.watersprings.org/pub/id/draft-moriarty-acme-client-01.html](http://www.watersprings.org/pub/id/draft-moriarty-acme-client-01.html)

<h2 id="cloud-provider-support">Cloud Provider Support</h2>




* AWS:
  [https://aws.amazon.com/elasticloadbalancing/features/](https://aws.amazon.com/elasticloadbalancing/features/)
* GCP:
  [https://cloud.google.com/appengine/docs/flexible/go/using-websockets-and-session-affinity](https://cloud.google.com/appengine/docs/flexible/go/using-websockets-and-session-affinity)
* Azure:
  [https://docs.microsoft.com/en-us/azure/application-gateway/application-gateway-websocket](https://docs.microsoft.com/en-us/azure/application-gateway/application-gateway-websocket)

<h2 id="other">Other</h2>




* [Websocket Load Balancing](https://pdf.sciencedirectassets.com/280203/1-s2.0-S1877050919X0006X/1-s2.0-S1877050919303576/main.pdf?X-Amz-Security-Token=IQoJb3JpZ2luX2VjEI3%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLWVhc3QtMSJHMEUCIAhC7%2Bztk8aH29lDsWYFIHLt97kwOE4PoWkiPfH2OTQwAiEA65oLMq1RhzF6b5pSixhnPVLT9G2iKkG145XtdpW4d4IqgwQIpv%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FARAEGgwwNTkwMDM1NDY4NjUiDDtEVrp4vXmh0hvwWyrXAxnfLN4%2BsMMF7wxoXOiBFQjn%2FJLpSLUIWghc87%2Bx2tbvdCIC%2BQV4JCY9rOK3p9rogqh9yoI2yem4SHASzL%2BQUQMOiGWagk%2FzyCNdS0y%2FLzHkKDahvRMJGKxWeXErbsuvPCufnbDpNHmKD0vnT5sqpOoM64%2FJVxvd9QYx48xasNMtXZ8%2BFm9wPpNQnsWSEZKYiOKLaLfnATzcXADJmOCTVQbwZoT4%2BFKWcoujBxSBHE9kw7S749ywQ9bOtgNWid5R2dj0z%2Br6C63SnBS3IdMSZ2qO4H3XTYY5pbfNCfR57zKIdwyp3zLJr5%2BtTEz1YR9FXwWF9niDEr0v2qu%2FlL7%2BGHsak8UQ4hZ0BFlZtcIRNW1lpZd9bNSINb3d6MnGeYrkhxQVP0KcZsowP9672IYzuMD4nK1X4Hv7bMqeO7ojuSf%2F2ND9NXn0Ldr%2BX0lzESv10LyhElCGfFJ4EZjIxYOKZdee1Zc1USdj1kNx1OC0cefIN1ixiA0OIbtWVz1lI6n1LYpngeUYngGP0ZFb%2Br%2FbleC3WarDHWIn4NNjI1aQW3P9fTmKEan3b3skRIBbwM8%2FrwRJGYQ03JaCKuU4xbogz9uEL%2BbpJ1SB7En8pS8xuSiE1kzvnsF0FTCEvMSIBjqlAadtZOgWRUk2FxdoYsCK43DYqD6zjbDrRBfyIXTJGlJYKt5iR3SCi8ySacO1aPZhah9ir179nYi5dVYnf5c6%2Fe8Q5Mo1uRtisouWJZSjAOhmRY7a76fSqyHwj088aI5t1pcempNCOnsM4SfyrZJ9UE%2FKfb5YsJ71VwRPZ%2BXZ%2FvZnQlW7e6NJqWswhre0pQftkShN%2BbpE%2FTzusekzm6q3w6b3ynUN8A%3D%3D&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Date=20210809T134614Z&X-Amz-SignedHeaders=host&X-Amz-Expires=299&X-Amz-Credential=ASIAQ3PHCVTY2T5F5OYZ%2F20210809%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Signature=6098b604ebac38723d26ae66e527b397312a6371ad19e1a4fbfe94ca9c61e1a9&hash=ebd5b943d3aff77c6bfb8853fab1598db53996f5f018d688364a41dd71c15d92&host=68042c943591013ac2b2430a89b270f6af2c76d8dfd086a07176afe7c76c2c61&pii=S1877050919303576&tid=spdf-3c0a3a1a-bd3b-40d0-af0d-48a46859c89a&sid=d21b79c59bbb0348b79945c084cc3b66983agxrqa&type=client)

--

Copyright The OpenTelemetry Authors

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
