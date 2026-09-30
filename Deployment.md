# Deployment

CoiPond splits into a control plane and worker nodes, and the control plane needs no Incus of its own. This page covers where each piece runs.

## Control Plane

The control plane needs no Incus of its own, so it can run on a bare host, in Docker, or - the cleanest option, since every deployment already has coi - as a **persistent coi container** launched directly from the checked-in `deploy/control-plane.toml` profile (which also publishes the ports the fleet needs).

## Worker Nodes

Worker nodes each run the per-node agent, a Solid Queue worker scoped to their queues, and a Falcon web process for terminal streaming.

For the full process layout (`Procfile.controlplane`, the per-node processes) and the coi-container control-plane setup, see [Running Across Multiple Machines](Multiple-Machines).
