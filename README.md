# The Bound Catalog: Why a Conversation's Tool List Belongs to the Host, Not the Model

**John Majerus**
*Draft — October 2026*

---

## Abstract

A Model Context Protocol host shows a model a list of tools, then keeps that list for the life of the conversation. The server, and any portal that stores a copy of the list, can change a name, a description, or an input schema afterward. The model goes on reasoning from the list it was shown. The result is a confident agent following a contract the server no longer publishes. Current practice answers this with a refresh control, a new chat, and, where a server is careful, a tool whose description quotes a catalog id and whose live result reports whether that id still matches. These are workarounds. They leave the comparison to the model, or to the person who remembers the order of operations. We argue that the missing primitive is *tool-catalog binding*: the host records the generation of the tool list it put in front of the model, and it refuses the next turn when a later `tools/list` is a different generation. The protocol already has an optional notice that the list changed. It does not have a generation, and it does not require the host to treat a mismatch as the end of that conversation's authority to call tools.

---

## 1. Introduction

An MCP server advertises tools. Each tool has a name, a description, and an input schema. The host fetches that list, inserts it into the model's context, and the model chooses calls from it. A later `tools/call` runs against the server that is live at that moment. The description the model read does not.

For a conversation that starts and ends under one unchanged server, the split is invisible. The list the model was shown is the list the server still implements. The failure appears when the advertised contract changes and the conversation does not. A description is rewritten. A parameter becomes required. A tool is removed. The open conversation still contains the previous advertisement. The model will explain that advertisement with the same confidence it uses for a current one, because nothing in its context marks the text as expired.

The failure is easy to miss when the change is a description and the name and inputs stay put. The tool is still there. A call may still succeed. What changed is the meaning the model was given for the call. A host that diffs tool lists by name will see no addition and no removal.

Operators answer with procedure. Deploy the server. Synchronize any portal that stores a catalog. Refresh the client's tool list. Start a new conversation, and start it after the refresh, because a conversation keeps the list it was given at the start. A control named for refreshing the tool list updates the connector's copy. It does not replace the copy already loaded in an open conversation. The procedure works when every step is done in order. It is a convention, practiced by the person and then by the model, in place of a check the host could make itself.

This paper argues that a conversation must be bound to the tool catalog it was shown, and that the host is the component that holds both copies and can compare them. A tool the model may call, and a sentence in a system prompt telling it to call that tool, approximate the check. They do not locate it.

---

## 2. A Stale Description, Illustrated

Consider an authoring server whose tools are reached through a portal. The portal stores the server's tool list and serves that stored list to clients. Tool calls are proxied to the live server. A maintainer changes the description of one existing tool. The tool keeps its name and its inputs. The new sentence says that a review proposal is stored as a patch against the working copy and that the working copy is then restored. The sentence it replaces said that the proposal was filed against the published revision.

The server process is restarted on the commit that contains the new sentence. The portal's stored catalog is synchronized. The portal's own tool page shows the new paragraph in full. A client that has been connected since before the sync still quotes the old sentence. So does a conversation in a hosted assistant, after the connector's tool list is refreshed, after the connector is disconnected and connected again, and after a new conversation is opened in the wrong order: the conversation first, the refresh second.

Nothing in that sequence is a mystery once the copies are named.

1. The server defines the tools.
2. The portal stores a copy and can lag the server until a synchronization finishes. On Cloudflare's MCP portals, tools and prompts for a server using dynamic client registration are synchronized on a schedule of about two hours, or on demand. Resources are not stored; a resource read is fetched live. A tool description is not [Cloudflare One, MCP server portals](https://developers.cloudflare.com/cloudflare-one/access-controls/ai-controls/mcp-portals/).
3. The client stores a third copy, taken when the connector connects or when its tool list is refreshed.
4. The conversation keeps the copy it was given when it started. Refreshing the connector, disconnecting, and signing in again update the connector. They leave an open conversation on the list it already has.

A person who refreshes and then starts a conversation gets the new list. A person who starts the conversation and then refreshes does not. The second order is the one the controls invite. The refresh control reports success. The conversation, which is where the question is asked, continues to quote the old contract.

The same split applies to a local client that connected before the server changed. Its session still holds the old description. A new session, started after the client reloads the server, holds the new one.

Two further details decide how serious a description-only edit is.

The model does not experience the old sentence as a cache hit. It experiences it as the tool's definition. Asked what the tool does, it paraphrases the text in its context. Asked to use the tool, it plans from that text. The live call can still perform the new behavior. The plan was made for the old one.

Guidance that the server returns from a tool call is a different channel. In the authoring server used here, `get_authoring_guidance` and `get_workflow_guidance` run on the server and return its current text. That text is not the stored catalog. A conversation with a stale tool list can still be told the truth if it calls one of those tools and the host actually dispatches the call. The description of the tool it is about to call is not in that channel. The description is the catalog entry it already has.

---

## 3. Why the Model Cannot Be the Check

The obvious patch is a tool. Give it a description that quotes the catalog id under which it was published, and have the call return the id the server has now. The description is frozen with the conversation. The call is live. If the model passes the id it was shown, the server can reply that the interface is current, or that it is not, and can tell the model to stop.

That tool is worth building, and Section 6 describes one. It is not the primitive.

The model has to call it. A system prompt can say to call it before authoring, and the tool's own description can say the same thing. Both of those texts are part of the snapshot the conversation started with. A conversation that started before the tool existed has no such call. A conversation that has the tool can ignore the instruction, call the other tools first, or read a negative result and continue. Compliance is whatever the model does on that turn. An instruction the model usually follows is not a guarantee that a stale contract will not be used.

This is the same misplacement described for document integrity elsewhere: a duty the infrastructure can discharge is assigned to the agent, because the agent is the component that happens to be reading the stale text [The Integrity Burden](https://github.com/jmajerus/write-domain-scoping). The agent is a poor place for the duty. The host already fetched the list, already inserted it into the prompt, and is about to send another turn. The host can compare. The model can only be asked to.

A hand-bumped version string fails in a more ordinary way. If a description changes and the string is not bumped, the old conversation and the server still agree. The check goes silent. A hash of the other tools' names, descriptions, and input schemas moves on its own when the advertisement changes. The hash fixes the bookkeeping. It does not move the comparison out of the model.

---

## 4. What the Protocol Already Says

The protocol already treats the tool list as something that changes, and as something clients will cache.

A server that supports tools declares a `tools` capability. `listChanged` says whether the server will emit a notification when the available tools change. Servers that declare the capability must answer `tools/list` with the tools currently available to the requesting client. The specification encourages a stable order across requests when the set has not changed, because clients cache the list and because a stable list improves prompt-cache hit rates when tools are included in model context [MCP specification, Tools](https://modelcontextprotocol.io/specification/draft/server/tools).

When the list changes, a server that declared `listChanged` should send `notifications/tools/list_changed` to clients that have opened a subscription for that notification [MCP specification, Tools](https://modelcontextprotocol.io/specification/draft/server/tools); [Subscriptions](https://modelcontextprotocol.io/specification/draft/basic/patterns/subscriptions). The notification carries no body. It does not name a generation. It does not include the new list. A client that wants the new list calls `tools/list` again.

That is a change signal for a connected, subscribed client. It is the right signal for a process that is still on the wire when a tool is registered or a description is edited. Several gaps remain, and they are the gaps that produced Section 2.

The notification is optional at every step. The server may decline to advertise `listChanged`. The client may decline to subscribe. A client that receives the notification is not required by the specification to replace the tool definitions it has already given the model, or to stop a conversation that was started under the previous list. The specification's own reason for a stable list — prompt-cache hit rates — is a reason for hosts to keep the inserted list still.

A disconnected client never sees the notification. A conversation opened against a stored catalog sees whatever generation the store held at fetch time. A later synchronization of that store does not travel back along a subscription the client did not have open. The next `tools/list` returns the new generation only when something asks.

`initialize` also carries a server version. Reconnecting clients can read it. An open conversation does not re-read the handshake, and the version is whatever the server author bumped. It is not a hash of the advertisement, and it is not in the model's tool list.

None of these mechanisms records the generation that was shown to the model beside the definitions that were shown, or compares that generation to the one a later call would execute against.

---

## 5. The Missing Primitive: Tool-Catalog Binding

Tool-catalog binding gives the tool list a generation and makes the host enforce it.

**Generation.** Every `tools/list` result carries a generation for the advertisement it contains: the names, titles, descriptions, input schemas, and the other fields the host will show the model. The generation is a property of that payload. A server computes it from the advertisement. An intermediary that serves a stored copy forwards the generation it stored, and can also report the generation the upstream server has now when it knows that second value. Two identical advertisements have the same generation. A description edit produces a new one. The generation is not a marketing version and not a hand-bumped counter.

**Binding.** When a host inserts a tool list into a conversation, it records the generation of that list as part of the conversation's state. The binding is the host's. It is not a parameter the model is asked to echo.

**Check.** Before the host accepts another model turn that may call tools, it compares the bound generation with the generation of the list it would call against. A portal that serves a stored catalog is part of that comparison: the generation the conversation was bound to, the generation the portal would serve now, and, when the portal has it, the generation of the upstream server. Any mismatch ends the conversation's authority to call tools.

**Stop.** The host does not silently rewrite the tool definitions under a turn the model has already planned. It stops. The person refreshes, and the next conversation is bound to the new generation. A control that refreshes the connector's tool list invalidates every conversation bound to the previous generation, or it does not claim to have refreshed the tool list those conversations are using.

The primitive has a narrow consequence and a wide one.

The narrow consequence is that a description-only edit becomes visible. The host does not have to guess whether the edit was a typo or a change in meaning. The description is part of the contract the model was given, so a new description is a new generation. Text that a tool returns when called — authoring guidance, a record fetched by id — stays out of the generation. That text is live. Putting it in the catalog generation would force a new conversation for a correction the next call would already return.

The wide consequence is where the duty sits. The model is not told to notice that its tools have expired. The host, which inserted them, notices. A stale catalog becomes a failed turn with a stated reason, in the same way a stale `If-Match` validator fails a write instead of asking the client to remember the version it saw [RFC 9110, Validator Fields](https://www.rfc-editor.org/rfc/rfc9110.html#name-validator-fields).

`notifications/tools/list_changed` remains useful. It tells a connected client to fetch the list now rather than at the next turn. Binding does not replace that notice. It defines what the host must do with the list it fetched, including a list fetched with no notice at all because the client was disconnected or was talking to a stored catalog.

---

## 6. The Working Approximation

Concept Clusters implements the approximation this paper refuses to treat as the guarantee. The authoring server registers `get_tool_catalog` after every other tool. The tool's description quotes a catalog id, a hash of the other tools' names, titles, descriptions, input schemas, and annotations. The call takes that id. The server compares it with the id it computed at registration and returns `interfaceCurrent`. A false result tells the model to stop: synchronize the portal if its tool page is old, refresh the connector's tool list, and start a new conversation. The server instructions say to make that call before authoring when the tool is in the list.

The hash excludes the catalog tool's own description, so the id embedded in that description is not an input to itself. A hand-bumped string was rejected because a forgotten bump leaves both sides in agreement. Guidance documents returned by `get_authoring_guidance` are not inputs to the hash. Their own major and minor version already distinguishes a rule change from a clarification, and a typo there does not bump it. A typo in a tool description does move the catalog id. That is accepted. The stale contract in Section 2 was a description edit.

The approximation has the limits Section 3 names. It appears in a conversation only after a client has refreshed and a new conversation has started with the tool in its list. Until then there is nothing to call. After that, the check runs if the model calls the tool and honors the result. The server can say stop. It cannot make the host stop.

The implementation is the production slice, in the same sense that a column partition is a production slice of write-domain scoping. It gives a willing model a way to detect a stale list on this server. It is evidence for the primitive, not a substitute that other servers should copy as their design.

---

## 7. A Counter-Argument: The Transcript Should Stay Coherent

The defense of the current behavior is real. Tool definitions are part of the prompt. Earlier turns in a conversation were produced against a particular list. Replacing that list in the middle of the conversation makes later turns see different tools from earlier ones. A model that planned a call against the old description, then receives a new description before the call, is in a worse state than a model with a consistently old list. Prompt caches, which the specification cites as a reason for a stable tool order, break when the prefix changes. Hosts that freeze the list for the life of the conversation are protecting coherence.

Binding agrees with the freeze. It disagrees with what the freeze is allowed to outlive.

A conversation bound to generation G is coherent under G. Calls made under that binding were chosen against G's descriptions and schemas. When the server, or the portal in front of it, moves to generation H, the coherent act is to stop calling tools under G. Continuing under G is coherent only inside the transcript. It is not coherent with the server. The model will defend G's description of a tool that H no longer implements, and a call it does make will be executed by H.

Hot-swapping H into the open conversation would buy freshness at the cost of the coherence the objection wants. Binding does not require that swap. It requires the host to end the old binding. The next conversation is bound to H from its first turn. The transcript of the old conversation remains a transcript of G, which is what it actually was. The person is told why the turn did not proceed.

The expensive part of today's procedure is not the existence of a new conversation. It is discovering, by questioning the model and comparing its answer to a dashboard, that the new conversation was required and was opened too early. A host that knows both generations can say so before the model has paraphrased the old one.

---

## 8. Intermediaries That Store a Catalog

A portal that stores tools and prompts is a second server from the client's point of view. The client does not know that `tools/list` was answered from a synchronization rather than from the process that will handle `tools/call`. The protocol's change notification does not cross that gap. The upstream server can emit `notifications/tools/list_changed` on its own connections. The portal's stored copy moves when a scheduled or manual synchronization finishes, which is a different event, often with no client attached.

Binding makes the intermediary visible without forbidding it. The portal's `tools/list` carries the generation of the advertisement it is serving. That generation changes when a synchronization installs a new advertisement, and when an operator overrides a description at the portal. Cloudflare's portals already store such overrides and mark a modified tool in the dashboard; clients then receive the override rather than the upstream sentence [Cloudflare One, MCP server portals](https://developers.cloudflare.com/cloudflare-one/access-controls/ai-controls/mcp-portals/). An override is a different contract. It deserves its own generation, or a client cannot tell a portal edit from the server definition it read last week.

A portal that can reach the upstream server can also return the upstream generation next to the generation it is serving. The host then distinguishes "the portal is behind the server" from "this conversation is behind the portal." Both are stops. They are different repairs: synchronize the portal, or refresh the client and start a new conversation. Today's stale conversation looks the same in both cases, and the person inspects a dashboard to tell them apart.

Code paths that strip descriptions and schemas out of the initial list, and refill them through a later search tool, are a form of the same binding problem. The definitions the model eventually sees are still a generation of an advertisement. Whenever those definitions are inserted, the host binds that generation. A search result that returns a tool definition from an older stored catalog is a stale bind, even if the conversation's original minimized list has not changed.

---

## 9. Related Work

**HTTP validators.** RFC 9110 defines validators, including `ETag`, so a cache and an origin can name a representation, and conditional requests so a client can require that the representation it saw is still current. `If-Match` fails the request when the validator does not match, rather than applying the request to a different representation [RFC 9110, Validator Fields](https://www.rfc-editor.org/rfc/rfc9110.html#name-validator-fields). Tool-catalog binding is that pattern for the tool list a model was shown. The conversation is the client that saw a representation. The next turn is the conditional request. The missing piece in MCP is the validator on `tools/list` and the host's obligation to send it.

**The MCP list-changed notification.** Section 4 describes the specification's existing notice. It is the closest mechanism, and it is a push hint to subscribed connections. It does not name a generation, it does not survive disconnection, and it does not say that a conversation whose inserted tools are of a previous generation must stop. The specification's caching guidance explains why hosts keep a fetched list stable. Binding is the condition under which that stability is still honest.

**Write-domain scoping.** *The Integrity Burden* argues that an agent must not be left to preserve parts of a document it does not own, and that the enforcement belongs in the infrastructure [The Integrity Burden](https://github.com/jmajerus/write-domain-scoping). The present paper is the same shape of argument at a different boundary. There, the agent is handed a document and asked to reproduce what it should not touch. Here, the agent is handed a tool contract and asked to notice when the contract has expired. In both cases the workaround is a convention the agent is told to follow, and the primitive is a check the host or the server can perform without the agent's cooperation. The primitives are not the same one. Scoping a write does not tell a host that its tool list is stale. Binding a catalog does not tell a server which fields an agent may change.

---

## 10. Conclusion

A tool list is the contract under which a model chooses calls. Hosts copy that contract into a conversation and then leave it there. Servers change the contract. Portals store an older copy of it. The model reports the copy it was given.

The protocol can already tell a subscribed client that the list changed. It cannot tell a conversation which generation it is bound to, and it does not require the host to stop when the generation moves. Refresh controls, new conversations, and a catalog tool that asks the model to compare ids are how operators and servers approximate that stop today.

Tool-catalog binding is the stop itself. The list carries a generation. The host records the generation it showed the model. A later list with a different generation ends that conversation's authority to call tools. The next conversation is bound to the new list from the start. The model is left with the duty it can actually discharge: choosing calls from the contract it was just shown.

---

*The author developed this argument while changing the tool catalog of the [Concept Clusters](https://github.com/jmajerus/concept-clusters/) authoring server and watching clients continue to quote the previous descriptions. The working approximation in Section 6 is `get_tool_catalog` in that server. This paper was drafted with the assistance of Grok. The ideas are the author's own.*
