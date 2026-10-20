

# Harness Container Deploy

## Purpose
Deploy a web service onto a harness (Hermit Control) container card the way this stack expects, and diagnose public-port reachability by failure mode instead of guessing.

## Workflow
1. **Confirm before creating.** Card creation, deletes, and driving a card's agent need Jim's explicit go-ahead; read-only checks (consoles, health endpoints, downloads) do not. For a substantive build, present the scheme first and get sign-off before touching anything.
2. **Create the card in order.** On the harness page, fill in the container name and click create — harness then automatically establishes the frp mapping. Only afterwards, write the requirements on the new card; the card's own agent deploys the service to container port 8082. Do not reorder these steps.
3. **Move files via the OCS relay.** Upload the build with `curl --upload-file <local-file> https://obs.dimond.top/<filename>` (PUT to `/{filename}`; returns 201 and the file URL), then have the container's agent download it inside the container with `curl` and verify with `md5sum`. Direct browser-task file transfer has hit authorization timeouts — use the relay instead of retrying that route.
4. **Drive the container through its card.** Use the card's send-message channel to run the container's own agent. The gpussh web terminal reached from the browser environment only shells into its own container, and host SSH from there fails authentication.
5. **Verify in layers, inside out.** Health check inside the container, then through the container network, then via the docker bridge gateway (e.g. `172.30.0.1:<port>`), then from the public internet at `http://dimond.top:<port>/`. The port mapping can work while frp is down, so a green health check does not prove public reachability.
6. **Diagnose the public port by failure mode.** Check the frps listeners on the host before blaming anything else:
   - Connection refused on `dimond.top:<port>` with no frps listener for that port: the frp mapping was never created — harness auto-frp didn't fire for this card.
   - Timeouts while the frps listener exists: the backend forwarding target is broken (the service-side endpoint is gone), not the mapping itself.
   - Never attribute it to the host firewall without checking the listeners first.
7. **Confirm the deployed file is actually new.** `md5sum` the file in the container against the local source; never present a re-deployed file as new without verifying the hash changed.

## Output Contract
Report the card/container name, the container port and public URL, reachability per verification layer, the `md5sum` match, and any infrastructure fault left open (e.g. a broken frp backend) flagged as Jim's decision — do not re-ask him about a call he already owns.

## Operating Rules
1. Stack choice: harness is the only stack — container cards go to the harness control plane (`localhost:19080` via the SSH host, ports 19081+), files to ocs/obs. The hermit instance (old machine, being retired per Jim 2026-10-20) is never used for new cards.
2. Follow the container norm: webapp + API, service on container port 8082, images and artifacts on OCS/obs, metadata in container-side storage.
3. When a container API call returns no response, treat it as unknown, not failed: retry once, then read the record back to confirm; never leave orphan files a later run cannot reconcile.
4. Keep this skill to general, repeatable method. Record per-deployment specifics (ports, app names, one-off workarounds) in memory or the task record, not here.
## Available connectors

Use `muse.connector_search` to find the current connector docs before following
recovered instructions.
