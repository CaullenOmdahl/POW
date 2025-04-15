# Remote Production Suite Gameplan

## Project Goal

Adapt the Wolf streaming server into a cloud-based remote production suite hosted on Vultr (Singapore region) for live events primarily in Southeast Asia (SEA). This aims to replace the current workflow of deploying laptops/NUCs on-site.

## Core Architecture

*   **Foundation:** Utilize Wolf's existing capabilities for containerization (Docker), low-latency streaming (Moonlight/Sunshine), and multi-session management.
*   **Host:** Vultr cloud instances (likely GPU-enabled).
*   **Client Access:** Operators connect via Moonlight clients.

## User Experience Goals

*   **Simplicity:** The entire process, from requesting an instance to connecting and operating, must be intuitive for users unfamiliar with cloud infrastructure, Linux, or Docker.
*   **Minimal Configuration:** Operators should not need to perform complex setup tasks within the remote environment (e.g., installing software, configuring networking).
*   **Seamless Reconnection:** Disconnections should be handled gracefully, allowing operators to reconnect to their exact previous state with minimal effort.
*   **"It Just Works" Peripherals:** Connecting required peripherals like Stream Decks should require minimal operator action.
*   **Abstraction:** Hide the underlying complexity of Vultr, Wolf, Docker, and networking from the end-user.

## Required Changes & Features (Wolf & Container Environment)

1.  **Software Stack:**
    *   **Standardize on Linux:** Migrate from Windows-based tools (like vMix) to Linux-native alternatives.
        *   **Primary Tool:** OBS Studio.
    *   **Pre-built Docker Images:** Create images containing *everything* needed for a specific production role. Goal: zero setup required by the operator *inside* the container. Components include:
        *   **Desktop Environment:** A lightweight Linux desktop environment (e.g., GNOME, potentially themed to resemble Pop!_OS for familiarity).
        *   **Remote Desktop Server:** VNC Server (e.g., `tigervnc-standalone-server`) configured to start the desktop session automatically.
        *   **OBS Studio:** The core application.
        *   **Essential OBS Plugins:** Pre-install necessary plugins (e.g., `obs-ndi`, advanced scene switchers, specific transition plugins, audio processing plugins, potentially others needed to mimic vMix features).
        *   **Default OBS Configuration:** Include pre-configured OBS scene collections, profiles, and UI layouts designed to provide a starting point familiar to vMix users.
        *   **Elgato Stream Deck control software:** (e.g., `streamdeck-ui` or similar).
        *   **USB-over-IP server component:** (e.g., VirtualHere server).
        *   **Other necessary production utilities.
    *   **Wolf App Configuration:** Update Wolf's application management (ref: `docs/modules/apps/pages/index.adoc`) to launch the *desktop session* (likely via a script that starts the VNC server or connects to it) which contains the pre-configured OBS, rather than launching OBS directly. Present a simple, user-friendly list of available production roles/images for selection during launch.

2.  **Session Persistence:**
    *   **Modify Wolf Core:** Adapt session management (`src/moonlight-server/state/sessions.cpp`, potentially API in `src/moonlight-server/api/api.hpp`) so that Docker containers persist even if the Moonlight client disconnects.
    *   **Reconnection Mechanism:** Implement a *seamless* way for operators to reconnect. Ideally, the Moonlight client simply reconnects, and Wolf automatically routes them to their existing persistent session without extra steps.

3.  **Peripheral Support / Hardware Integration:**
    *   **Stream Deck:**
        *   **USB Forwarding:** Integrate a USB-over-IP solution designed for simplicity. Goal: Operator plugs in Stream Deck, runs a simple client-side utility (if necessary), and it's automatically available in the remote session. Minimize configuration.
        *   **Control Software:** Pre-configure the chosen Stream Deck software (e.g., `streamdeck-ui`) inside the container to interact with OBS with sensible defaults.
    *   **TY-REW580 Slow Motion Switcher:**
        *   **Control Method:** Determine how the switcher is controlled (Dedicated software? Network API/protocol? Web UI? Physical connection like USB/Serial?).
        *   **Integration:** Plan how the remote OBS instance/desktop environment will interact with the switcher based on its control method and physical location (likely requires network access or device forwarding).
    *   **TY-REW580 Control & Integration:**
        *   How is the TY-REW580 controlled? (Specific software? Network protocol/API? Web UI? USB/Serial?)
        *   Is there Linux-compatible control software or a documented network API?
        *   Where will the TY-REW580 physically reside (event site, operator location)?
        *   How will the cloud instance communicate with it (Direct network access? VPN? Device forwarding like USB-over-IP?)?
        *   How does it integrate into an OBS workflow (Plugin? Separate application control?)?

4.  **Ingest/Egress:**
    *   **Ingest:** Assume SRT streams from on-site hardware encoders (e.g., Haivision) as input sources (configured within OBS).
    *   **Egress:** Configure OBS within containers to output the final program via SRT to designated destinations (e.g., Haivision decoders).

5.  **Shared Storage:**
    *   **Mounting:** Configure Docker (via Wolf or orchestration layer) to mount a shared network filesystem (e.g., Vultr Block Storage via NFS) into each container *transparently* to the user.
    *   **Directory Structure:** Container entrypoint scripts should create a unique subdirectory within the shared storage based on the instance/session ID. Make this easily accessible (e.g., a desktop shortcut inside the container).

## Dependency Management Strategy

*   **Goal:** Ensure long-term stability and prevent unexpected breakages due to updates in external software components (OBS, plugins, drivers), crucial for a good UX.
*   **Primary Method: Version Pinning:**
    *   Within the pre-built Docker images, install *specific, fixed versions* of all key software (OBS Studio, `obs-ndi`, Stream Deck control software, USB-over-IP components, base OS packages).
    *   Do *not* use `latest` tags or commands that automatically pull the newest version during image build. Reference specific version numbers or commit hashes.
    *   Updates to these pinned versions should be a deliberate process involving thorough testing in a staging environment before deployment to production images.
*   **Secondary Method: Forking (If Necessary):**
    *   For critical components identified as particularly unstable or prone to breaking changes (even with pinning), consider creating private forks of their repositories.
    *   This provides absolute control but significantly increases maintenance overhead (managing the fork, selectively merging upstream updates).
    *   Evaluate this on a case-by-case basis *only if* version pinning proves insufficient during testing.

## External Orchestration Layer (Separate System - Not part of Wolf codebase)

*   **Simplified UI:** This layer provides the *primary user interface*. It must be extremely simple:
    *   Select Production Role/Image (e.g., "Director Station", "Graphics Station").
    *   Select Duration (e.g., "24 hours", "48 hours").
    *   Click "Launch".
*   **Abstraction:** Handle all Vultr API interactions, instance configuration, storage setup, and Wolf startup parameters automatically based on the user's simple selections. Hide all infrastructure details.
*   **Provisioning:** Automate creation and termination of Vultr instances using the Vultr API.
*   **Expiration:** Implement user-defined expiration times for instances.
*   **Configuration:** Pass necessary configuration (e.g., selected tools/image, session details) to Wolf upon instance startup.
*   **Storage Management:** Handle the creation/attachment/detachment of shared storage volumes.
*   **API:** Provide an interface for users to request/manage production instances (this API is used by the Simplified UI).
*   **(Optional) Auto-Scaling:** Monitor load and potentially adjust the number of instances (may not be required if workload is predictable).

## Open Questions & Areas for Investigation

*   **Full Software Inventory:** What specific Windows software (graphics, audio processing, utilities) beyond vMix needs a Linux equivalent?
*   **Linux Equivalents:** Identify and vet specific Linux-native replacements for all required software.
*   **USB-over-IP Reliability & UX:** Test performance, stability, and *ease of setup/use* for the chosen USB-over-IP solution (e.g., VirtualHere) with Stream Decks from an operator perspective. How many clicks? What needs installing on the operator's machine?
*   **Latency Validation:** Conduct real-world latency tests:
    *   Event Site (SEA) <-> Vultr Singapore (SRT Ingest)
    *   Operator Location (SEA) <-> Vultr Singapore (Moonlight Control)
    *   Vultr Singapore <-> Egress Point (SRT Output)
*   **Stream Deck Integration UX:** Confirm *intuitive* control of OBS from Stream Deck via the chosen Linux software and USB forwarding. Does the default configuration make sense?
*   **Orchestration UI/UX Design:** Design and prototype the user interface for the external orchestration layer.
*   **Security:** Develop a comprehensive security plan (network access, API security, data protection).
*   **Monitoring:** Define key metrics and implement monitoring for stream quality, instance health, resource usage, and network performance.
*   **Cost:** Estimate Vultr costs (GPU instances, block storage, bandwidth).
*   **Workflow Adaptation & Training:** How significantly will operator workflows need to change? What training materials are needed for the new tools and remote workflow?
*   **User Testing:** Plan for iterative testing with target users (non-technical operators) throughout the development process. for production work?
*   **Error Handling & Support:** How will users report issues? How will common problems (e.g., connection issues, software glitches) be communicated and resolved simply?ions, multi-view setups, audio routing, replay features if needed).

*   **OBS Default Configuration Design:** How should the default OBS scenes, profiles, hotkeys, and UI layout be designed to best ease the transition for vMix users?*   **Specific OBS Plugins:** Identify the *complete list* of OBS plugins required to replicate the essential vMix workflows used by the team (e.g., specific transitions, multi-view setups, audio routing, replay features if needed).*   **Desktop Environment Performance:** What is the performance overhead (CPU, GPU, RAM, latency) of running a full desktop environment + VNC server inside the container and streaming it via Moonlight? Is it sufficiently responsive for production work?*   **High-Risk Dependencies:** Which specific components (plugins, drivers like Stream Deck software, USB-over-IP) have a history of instability or breaking changes that might warrant closer monitoring or consideration for forking early on?*   **OBS Default Configuration Design:** How should the default OBS scenes, profiles, hotkeys, and UI layout be designed to best ease the transition for vMix users?