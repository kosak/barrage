---
title: RPC Interface
---

<!---
  Copyright 2020 Deephaven Data Labs

  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->

The Barrage extension works by sending additional metadata via the `app_metadata` field on `FlightData`. This metadata
is used to communicate the necessary additional information between server and client. These types are flatbuffers, so
that we may more easily lift the `app_metadata` into the `RecordBatch` flatbuffer once Arrow supports byte-array
metadata, at that layer.

The main subscription mechanism is initiated via a `DoExchange`. The client sends a SubscriptionRequest (or as many as
they like) and the server sends barrage updates to satisfy their subscription's requirements.

## Flat Buffer Definitions

<!-- fbs, not ts, but we don't have a syntax highlighter for that -->

```ts
// Copyright 2020 Deephaven Data Labs
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//   http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

namespace io.deephaven.barrage.flatbuf;

enum BarrageMessageType : byte {
  /// A barrage message wrapper might send a None message type
  /// if the msg_payload is empty.
  None = 0,

  /// for session management (not-yet-used)
  NewSessionRequest = 1,
  RefreshSessionRequest = 2,
  SessionInfoResponse = 3,

  /// for subscription parsing/management (aka DoPut, DoExchange)
  BarrageSerializationOptions = 4,
  BarrageSubscriptionRequest = 5,
  BarrageUpdateMetadata = 6,
  BarrageSnapshotRequest = 7,
  BarragePublicationRequest = 8,

  // enum values greater than 127 are reserved for custom client use
}

/// The message wrapper used for all barrage app_metadata fields.
table BarrageMessageWrapper {
  /// Used to identify this type of app_metadata vs other applications.
  /// The magic value is '0x6E687064'. It is the numerical representation of the ASCII "dphn".
  magic: uint;

  /// The msg type being sent.
  msg_type: BarrageMessageType;

  /// The msg payload.
  msg_payload: [byte];
}

/// Establish a new session.
table NewSessionRequest {
  /// A nested protocol version (gets delegated to handshake)
  protocol_version: uint;

  /// Arbitrary auth/handshake info.
  payload: [byte];
}

/// Refresh the provided session.
table RefreshSessionRequest {
  /// this session token is only required if it is the first request of a handshake rpc stream
  session: [byte];
}

/// Information about the current session state.
table SessionInfoResponse {
  /// this is the metadata header to identify this session with future requests; it must be lower-case and remain static for the life of the session
  metadata_header: [byte];

  /// this is the session_token; note that it may rotate
  session_token: [byte];

  /// a suggested time for the user to refresh the session if they do not do so earlier; value is denoted in milliseconds since epoch
  token_refresh_deadline_ms: long;
}

/// There will always be types that cannot be easily supported over IPC. These are the options:
///   Stringify (default) - Pretend the column is a string when sending over Arrow Flight (default)
///   JavaSerialization   - Use java serialization; the client is responsible for the deserialization
///   ThrowError          - Refuse to send the column and throw an internal error sharing as much detail as possible
enum ColumnConversionMode : byte { Stringify = 1, JavaSerialization, ThrowError }

table BarrageSubscriptionOptions {
  /// see enum for details
  column_conversion_mode: ColumnConversionMode = Stringify;

  /// Deephaven reserves a value in the range of primitives as a custom NULL value. This enables more efficient transmission
  /// by eliminating the additional complexity of the validity buffer.
  use_deephaven_nulls: bool;

  /// Explicitly set the update interval for this subscription. Note that subscriptions with different update intervals
  /// cannot share intermediary state with other subscriptions and greatly increases the footprint of the non-conforming subscription.
  ///
  /// Note: if not supplied (default of zero) then the server uses a consistent value to be efficient and fair to all clients
  min_update_interval_ms: int;

  /// Specify a preferred batch size. Server is allowed to be configured to restrict possible values. Too small of a
  /// batch size may be dominated with header costs as each batch is wrapped into a separate RecordBatch. Too large of
  /// a payload and it may not fit within the maximum payload size. A good default might be 4096.
  batch_size: int;

  /// Specify a maximum allowed message size. Server will enforce this limit by reducing batch size (to a lower limit
  /// of one row per batch). If the message size limit cannot be met due to large row sizes, the server will throw a
  /// `Status.RESOURCE_EXHAUSTED` exception
  max_message_size: int;
}

/// Describes the subscription the client would like to acquire.
table BarrageSubscriptionRequest {
  /// Ticket for the source data set.
  ticket: [byte];

  /// The bitset of columns to subscribe. If not provided then all columns are subscribed.
  columns: [byte];

  /// This is an encoded and compressed RowSet in position-space to subscribe to. If not provided then the entire
  /// table is requested.
  viewport: [byte];

  /// Options to configure your subscription.
  subscription_options: BarrageSubscriptionOptions;

  /// When this is set the viewport RowSet will be inverted against the length of the table. That is to say
  /// every position value is converted from `i` to `n - i - 1` if the table has `n` rows.
  reverse_viewport: bool;
}

table BarrageSnapshotOptions {
  /// see enum for details
  column_conversion_mode: ColumnConversionMode = Stringify;

  /// Deephaven reserves a value in the range of primitives as a custom NULL value. This enables more efficient transmission
  /// by eliminating the additional complexity of the validity buffer.
  use_deephaven_nulls: bool;

  /// Specify a preferred batch size. Server is allowed to be configured to restrict possible values. Too small of a
  /// batch size may be dominated with header costs as each batch is wrapped into a separate RecordBatch. Too large of
  /// a payload and it may not fit within the maximum payload size. A good default might be 4096.
  batch_size: int;

  /// Specify a maximum allowed message size. Server will enforce this limit by reducing batch size (to a lower limit
  /// of one row per batch). If the message size limit cannot be met due to large row sizes, the server will throw a
  /// `Status.RESOURCE_EXHAUSTED` exception
  max_message_size: int;
}

/// Describes the snapshot the client would like to acquire.
table BarrageSnapshotRequest {
  /// Ticket for the source data set.
  ticket: [byte];

  /// The bitset of columns to request. If not provided then all columns are requested.
  columns: [byte];

  /// This is an encoded and compressed RowSet in position-space to subscribe to. If not provided then the entire
  /// table is requested.
  viewport: [byte];

  /// Options to configure your subscription.
  snapshot_options: BarrageSnapshotOptions;

  /// When this is set the viewport RowSet will be inverted against the length of the table. That is to say
  /// every position value is converted from `i` to `n - i - 1` if the table has `n` rows.
  reverse_viewport: bool;
}

table BarragePublicationOptions {
  /// Deephaven reserves a value in the range of primitives as a custom NULL value. This enables more efficient transmission
  /// by eliminating the additional complexity of the validity buffer.
  use_deephaven_nulls: bool;
}

/// Describes the table update stream the client would like to push to. This is similar to a DoPut but the client
/// will send BarrageUpdateMetadata to explicitly describe the row key space. The updates sent adhere to the table
/// update model semantics; thus BarragePublication enables the client to upload a ticking table.
table BarragePublicationRequest {
  /// The destination Ticket.
  ticket: [byte];

  /// Options to configure your request.
  publish_options: BarragePublicationOptions;
}

/// Holds all of the rowset data structures for the column being modified.
table BarrageModColumnMetadata {
  /// This is an encoded and compressed RowSet for this column (within the viewport) that were modified.
  /// There is no notification for modifications outside of the viewport.
  modified_rows: [byte];
}

/// A data header describing the shared memory layout of a "record" or "row"
/// batch for a ticking barrage table.
table BarrageUpdateMetadata {
  /// This batch is generated from an upstream table that ticks independently of the stream. If
  /// multiple events are coalesced into one update, the server may communicate that here for
  /// informational purposes.
  first_seq: long;
  last_seq: long;

  /// Indicates if this message was sent due to upstream ticks or due to a subscription change.
  is_snapshot: bool;

  /// If this is a snapshot and the subscription is a viewport, then the effectively subscribed viewport
  /// will be included in the payload. It is an encoded and compressed RowSet.
  effective_viewport: [byte];

  /// When this is set the viewport RowSet will be inverted against the length of the table. That is to say
  /// every position value is converted from `i` to `n - i - 1` if the table has `n` rows.
  effective_reverse_viewport: bool;

  /// If this is a snapshot, then the effectively subscribed column set will be included in the payload.
  effective_column_set: [byte];

  /// This is an encoded and compressed RowSet that was added in this update.
  added_rows: [byte];

  /// This is an encoded and compressed RowSet that was removed in this update.
  removed_rows: [byte];

  /// This is an encoded and compressed RowSetShiftData describing how the keyspace of unmodified rows changed.
  shift_data: [byte];

  /// This is an encoded and compressed RowSet that was included with this update.
  /// (the server may include rows not in addedRows if this is a viewport subscription to refresh
  ///  unmodified rows that were scoped into view)
  added_rows_included: [byte];

  /// The list of modified column data are in the same order as the field nodes on the schema.
  mod_column_nodes: [BarrageModColumnMetadata];
}
```
