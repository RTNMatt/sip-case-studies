# **SIP over TLS Call Teardown Failure Caused by NAT, Contact Header, and Transport Assumptions**

## **Environment Overview**

### **Infrastructure Components**

* **SIP Endpoints**

  * Emergency call devices (headless endpoints, no keypad)

* **Customer SBC**

  * Carrier-grade SBC terminating SIP over TLS

  * Originates SIP over TLS on a non-default port and expects in-dialog requests to reuse the established transport connection

* **Emergency Services Provider (ESP)**

  * SIP proxy stack consisting of:

    * SIP application layer

    * SIP proxy (stateless for transport)

* **Network**

  * NAT present on the ESP side of the interconnect

  * SIP over TLS (TCP, non-default port 5060\)

  * No SIP ALG in path

### **Deployment Context**

* SBC hosted in a provider-managed environment

* ESP hosted in a cloud environment with source NAT present

* SIP over TLS used for signaling security

* Emergency calling scenario where call teardown and callback reliability are mandatory

---

## **Problem Statement**

### **Observable Symptoms**

* Emergency calls successfully established

* Audio path stable during active calls

* When the remote emergency operator terminated the call:

  * The SIP **BYE never reached the SBC**

  * The call leg remained active until timeout

* Emergency callback attempts failed because the endpoint was still seized

### **Impact**

* Endpoints unable to place or receive subsequent calls

* Emergency callback failure

* Regulatory and safety compliance risk

* Issue persisted intermittently and only surfaced during call teardown

---

## **Initial Hypotheses**

### **1\. TLS Version or Cipher Mismatch**

**Why it was reasonable**

* Packet captures showed TLS record-layer headers indicating TLS 1.0

* SBC was configured to require TLS 1.2+

**Why it was insufficient**

* ClientHello explicitly negotiated TLS 1.2 (TLS 1.0 record-layer wrapper is permitted by RFC 5246 Appendix E.1)

* No protocol\_version handshake\_failure alerts were generated (a TCP RST was observed instead)

* Failure occurred prior to TLS handshake enforcement, during peer association, when a new TLS connection was attempted solely for the BYE request and could not be mapped to a trusted peer profile.

---

### **2\. TLS Identity or SNI Mismatch**

**Why it was reasonable**

* New TLS connections were observed during BYE transmission

* No SNI was present in ClientHello

* SBC rejected connections before ServerHello

**Why it was incomplete**

* Did not explain *why* a new TLS session was created for BYE

* Pointed to a symptom, not the root cause

---

### **3\. SBC Policy Rejecting Mid-Dialog Connections**

**Why it was reasonable**

* SBC enforces strict peer identity validation

* Mid-dialog requests arriving on new transports can be rejected

**Why it was incomplete**

* SIP allows in-dialog requests on new connections

* Still did not explain why the existing connection wasn’t reused

---

## **Investigation & Evidence**

### **SIP Signaling Observations**

* INVITE, provisional responses, and 200 OK exchanged successfully

* Dialog established correctly

* When the remote side attempted call teardown:

  * ESP generated a BYE internally

  * BYE never appeared at the SBC

### **Packet Capture Findings**

* BYE generation triggered a **new TCP connection**

* New TLS ClientHello sent

* Connection reset by SBC prior to handshake completion

* No BYE ever transmitted over the original connection

### **Key SIP Details**

* The SBC originated SIP signaling on a configured SIP-over-TLS port (TCP 5060\)

* The ESP generated in-dialog BYE requests from a dynamically allocated source port

* The SBC included the source port in the Contact header of the initial INVITE

* During internal processing, the ESP did not consistently preserve the Contact header port when generating in-dialog requests

---

## **Root Cause Analysis**

### **Root Cause: Incorrect BYE Routing Due to Contact Header and NAT Interaction**

Two independent but related issues caused the failure.

---

### **Issue 1: SIP Responses Sent to Wrong Source Port (NAT \+ Missing rport)**

* SIP responses traversed a NAT boundary

* ESP sent responses based on the Via header or default port assumptions

* `rport` parameter was not consistently present

* SIP response routing across NAT was potentially ambiguous

**Effect**

* SIP response routing was fragile across NAT boundaries, though call establishment typically succeeded

* This condition did not directly cause call failure but contributed to overall signaling brittleness

---

### **Issue 2 (Primary): BYE Misrouted Due to Missing Contact Port**

* BYE is an **in-dialog request**, not a response

* `rport` does **not** apply to requests

* Per RFC 3261:

  * In-dialog requests are routed to the remote target derived from the Contact header

* The SBC included the source port in the Contact header of the initial INVITE

* During internal processing, the ESP did not consistently preserve the Contact header port when generating in-dialog requests

* The ESP attempted to send the BYE using the default transport assumptions

**Result**

* BYE sent to incorrect destination

* The ESP attempted to establish a new outbound connection to deliver the BYE

* New TLS handshake initiated

* The SBC rejected the new TLS connection before it could be mapped to a trusted peer, preventing the BYE from being processed.

* The BYE was never delivered to the SBC

---

### **Why TLS Appeared to Be the Problem**

TLS was not the root cause of the failure.

TLS-related failures occurred **because**:

* The BYE was misrouted

* A new connection was attempted unnecessarily

* The SBC could not associate the new connection with an existing dialog or trusted peer

TLS enforced correctness at the transport and trust boundary, exposing the underlying SIP routing issue rather than causing it.

---

## **Resolution**

### **Changes Implemented**

1. **Force `rport` in Via Headers**

   * Ensured SIP responses return to correct source port across NAT

2. **Rewrite Contact Header to Include Source Port**

   * Contact URI explicitly includes the received source port

   * Ensures correct routing of in-dialog requests (BYE)

### **Why This Worked**

* BYE now targets the correct port

* BYE is delivered over the existing connection

* No unplanned new TLS session is required

* Call teardown completes normally

* Endpoint becomes available for callbacks

---

## **Preventive Measures / Lessons Learned**

### **SIP and NAT**

* Always include `rport` when SIP crosses NAT boundaries

* Never assume default ports for TLS-based SIP dialogs

### **Contact Header Hygiene**

* Contact headers must include explicit ports when NAT or ephemeral ports are involved

* Missing ports silently break in-dialog requests

### **TLS Misdiagnosis Risk**

* TLS handshake failures can be **secondary symptoms**

* Always verify SIP routing behavior before blaming cryptography

### **Monitoring Improvements**

* Alert on dialogs exceeding expected duration

* Monitor for missing BYE delivery

* Track unexpected TLS handshakes mid-dialog

### **Design Lessons**

* Emergency calling requires deterministic teardown

* “Mostly works” signaling is unacceptable in safety-critical systems

* SIP correctness matters most at call teardown, not call setup

---

## **Summary**

This issue was not caused by TLS versions, cipher suites, or certificate handling.

It was caused by:

* Incorrect SIP routing assumptions across NAT

* Inconsistent preservation of Contact header port information across dialog state

* Resulting misdelivery of in-dialog BYE requests

The fix was purely protocol-correct SIP behavior.

This case demonstrates how transport security issues often surface deeper signaling flaws, and why packet-level analysis remains essential in modern cloud and carrier environments.

