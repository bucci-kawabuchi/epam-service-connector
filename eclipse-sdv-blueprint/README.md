# EV Range Extender Blueprint

This blueprint demonstrates an **open, use-case-driven application lifecycle** for Software-Defined Vehicles (SDVs). The **EV Range Extender is the example application**; the **main deliverable is the reusable integration pattern** connecting application development, vehicle APIs, cloud-to-edge deployment, distributed communication, and validation.

The blueprint combines Eclipse SDV projects and open interfaces so that individual components can be replaced or extended without changing the overall lifecycle concept. The hosted environments used by this repository are reference implementations of parts of that lifecycle, not the blueprint itself.

## Table of Contents

- [Blueprint Purpose](#blueprint-purpose)
  - [Eclipse Projects and Open Standards](#eclipse-projects-and-open-standards)
  - [Other Reference Components](#other-reference-components)
  - [Implementation Status](#implementation-status)
- [Demonstrated Use Case: EV Range Extender](#demonstrated-use-case-ev-range-extender)
  - [Use Case Flow](#use-case-flow)
  - [Vehicle Signals](#vehicle-signals)
- [Current Reference Implementation](#current-reference-implementation)
  - [Hosted Environments](#hosted-environments)
  - [Current Workflow and Validation Boundary](#current-workflow-and-validation-boundary)
- [Proposed Virtual-Prototyping Extension](#proposed-virtual-prototyping-extension)
- [Phase 1 Reference Implementation: QEMU](#phase-1-reference-implementation-qemu)
  - [Component Distribution](#component-distribution)
  - [System Setup Workflow](#system-setup-workflow)
  - [Run the Demo](#run-the-demo)
  - [Signal Flow and Internals](#signal-flow-and-internals)
  - [Troubleshooting](#troubleshooting)
- [Planned Phase 2: Physical Hardware](#planned-phase-2-physical-hardware)
- [Project Resources](#project-resources)

---

## Blueprint Purpose

The blueprint is intended to show how an SDV application can move through a coherent lifecycle while using Eclipse projects at the relevant integration points:

```text
Define the use case and VSS contract
        ↓
Develop and package the application
        ↓
Publish it to a lifecycle-management backend
        ↓
Deploy and run it on an edge target
        ↓
Exchange vehicle signals across compute domains
        ↓
Observe, validate, update, and repeat
```

It is not intended to prescribe one hosted portal, cloud service, hardware platform, or programming language. The current implementation selects concrete technologies to make the pattern reproducible. Future implementations can substitute equivalent components while retaining the same interfaces and lifecycle stages.

### Eclipse Projects and Open Standards

| Project or standard | Role in the blueprint | Status |
|---|---|---|
| [Eclipse AutoWRX](https://github.com/eclipse-autowrx) | Open-source implementation of the digital.auto development and prototyping environment; also provides the integration used to connect the running demo to the Playground dashboard | Used for the current dashboard integration; broader virtual-prototyping use is proposed |
| [Eclipse KUKSA](https://github.com/eclipse-kuksa) | Vehicle data broker and API for reading and writing vehicle signals | Implemented in Phase 1 |
| [Eclipse Zenoh](https://github.com/eclipse-zenoh) | Lightweight communication between the HPC and simulated ECU domains | Implemented in Phase 1 |
| [Eclipse Velocitas](https://github.com/eclipse-velocitas) | SDKs and vehicle-model tooling that can help keep Python prototypes and C++ implementations aligned to the same vehicle API | Proposed for the prototype-to-C++ workflow |
| [Eclipse S-CORE](https://github.com/eclipse-score) | Communication and middleware capabilities for the planned HPC-to-zonal integration | Planned for Phase 2 |
| [Eclipse ThreadX](https://github.com/eclipse-threadx/threadx) | RTOS for the planned microcontroller-based end-ECU layer | Planned for Phase 2 |
| [COVESA Vehicle Signal Specification](https://github.com/COVESA/vehicle_signal_specification) | Common semantic contract for vehicle signals across development and deployment environments | Used throughout the blueprint |

### Other Reference Components

| Component | Role in the current implementation |
|---|---|
| [AosCore](https://github.com/aosedge) | Open-source edge runtime and orchestrator deployed on the QEMU targets |
| AosCloud | Hosted reference backend for application registry, target configuration, and lifecycle orchestration |
| [QEMU](https://www.qemu.org/) | Virtualizes the two Linux compute targets used by Phase 1 |
| Hardware simulator in this repository | Supplies reproducible battery and cabin inputs to the deployed system |

### Implementation Status

| Scope | Status | What it demonstrates |
|---|---|---|
| Phase 1: QEMU reference implementation | Available | Build, registry publication, AosEdge deployment, distributed vehicle-signal flow, dashboard observation, and validation on two QEMU VMs |
| Playground virtual-prototype stage | Proposed | Python business-logic validation against a virtual vehicle before the production implementation is built and deployed |
| Phase 2: physical hardware | Under development | Migration of the reference architecture to HPC, zonal, and end-ECU hardware |

---

## Demonstrated Use Case: EV Range Extender

The EV Range Extender application monitors the traction battery State of Charge (SoC). When the SoC crosses configured thresholds, the application reduces non-essential energy consumption—such as HVAC and seat heating—while leaving critical driving and safety functions unaffected. The dashboard shows the resulting actuator state and estimated-range changes.

The example is intentionally understandable at the vehicle-feature level. Its purpose is to exercise the blueprint's software lifecycle and signal integration, not to provide a production-ready energy-management algorithm.

The current demo also does not make a functional-safety or mixed-criticality certification claim. Those concerns require target-specific architecture, isolation, assurance, and validation beyond this reference use case.

### Use Case Flow

| Step | Actor | Behaviour | Driver-visible result |
|---|---|---|---|
| 1. Monitor | EV Range Extender | Continuously reads battery SoC and related powertrain values | No action while charge remains above the configured thresholds |
| 2. Reduce load | EV Range Extender | At 50% SoC, turns off the HVAC fan; at 30%, also turns off seat heating/cooling | Cabin comfort functions are reduced in stages |
| 3. Report | EV Range Extender and dashboard | Publishes actuator states and updated estimated range | The driver can observe power-saving actions and their effect |

> **Why this matters for OEMs:** The use case demonstrates how an independently deployable vehicle application can observe shared vehicle data, coordinate functions across compute domains, and be updated through a managed lifecycle.

### Vehicle Signals

| Runtime location | VSS signal | Function | Use in the demo |
|---|---|---|---|
| VM1 | `Vehicle.Powertrain.TractionBattery.StateOfCharge.Current` | BMS | Triggers staged power-saving behaviour |
| VM1 | `Vehicle.Powertrain.TractionBattery.CurrentVoltage` | BMS | Provides battery voltage |
| VM1 | `Vehicle.Powertrain.TractionBattery.CurrentCurrent` | BMS | Provides battery current |
| VM2 | `Vehicle.Cabin.HVAC.AmbientAirTemperature` | HVAC ECU | Represents a cabin/HVAC value synchronized between domains |
| VM2 | `Vehicle.Cabin.Seat.Row1.DriverSide.Heating` | Seat ECU | Represents seat-heating state |
| VM2 | `Vehicle.Cabin.Seat.Row1.DriverSide.HeatingCooling` | Seat ECU | Is disabled as part of the second power-saving stage |

---

## Current Reference Implementation

The current implementation uses a hosted development portal and lifecycle backend to make the blueprint quick to access, while running the actual application and support services on QEMU-based edge targets.

### Hosted Environments

- **digital.auto Playground** is the convenient hosted environment used to edit and host the application definition, trigger the build and deployment flow, select the runtime, and visualize the running demo. [Eclipse AutoWRX](https://github.com/eclipse-autowrx) is the open-source implementation of digital.auto and is the appropriate basis for reproducing or extending these capabilities outside the hosted Playground.
- **AosCloud** is the hosted lifecycle-management backend used by this reference implementation for the application registry, target configuration, and deployment orchestration. It is one implementation choice within the blueprint rather than a required definition of the blueprint concept.
- **AosCore on QEMU** is where the current application is deployed and executed. The supporting BMS, HVAC, seat, and range services run across the two virtual machines and exchange data through KUKSA and Zenoh.

The repository currently documents and validates this specific combination. Alternative deployments using a local AutoWRX stack or another compatible lifecycle backend are possible extensions, but they are not part of the tested setup described below.

> **Reproducibility note:** The current deployment procedure requires access to the hosted AosCloud service. AosCloud is therefore treated here as a reference lifecycle backend, not as a mandatory Eclipse blueprint component. This repository does not yet provide or validate a fully self-hosted replacement for that backend.

**[Open the EV Range Extender in digital.auto Playground](https://playground.digital.auto/model/67f76c0d8c609a0027662a69/library/prototype/69ce30f438bb8e98f0af5ac8/view)**

### Current Workflow and Validation Boundary

```text
digital.auto Playground
  Edit/configure the application and trigger the C++ build
        ↓
AosCloud
  Store the package, configure the target, and orchestrate deployment
        ↓
AosCore on QEMU-VM-1 and QEMU-VM-2
  Run the application and supporting ECU services
        ↓
Hardware simulator + KUKSA + Zenoh
  Provide signals and connect the simulated vehicle domains
        ↓
digital.auto Playground dashboard
  Observe the behaviour of the deployed system
```

In the current setup, Playground enables code editing, code hosting, the build/deployment trigger, and dashboard visualization. **Functional validation of the EV Range Extender takes place after deployment on the QEMU reference environment; the application is not first executed and validated as a virtual prototype inside Playground.** This distinction is important when evaluating what the current blueprint demonstrates.

---

## Proposed Virtual-Prototyping Extension

A useful next step is to add an explicit virtual-prototype stage before the C++ application is packaged and deployed. This would make the lifecycle closer to the intended digital.auto development model while remaining a relatively small extension of the current setup:

1. Develop the initial vehicle-application logic in Python in digital.auto Playground.
2. Run it against an AutoWRX virtual vehicle environment, such as the [AutoWRX SDV Runtime](https://github.com/eclipse-autowrx/sdv-runtime), using simulated VSS inputs.
3. Validate thresholds, signal interactions, state transitions, and expected outputs before edge deployment.
4. Capture the validated VSS model, configuration, scenarios, and automated behavioural tests as the portable application contract.
5. Productionize the application in C++ using the [Eclipse Velocitas C++ SDK](https://github.com/eclipse-velocitas/vehicle-app-cpp-sdk) and common vehicle-model tooling where applicable.
6. Build and deploy the C++ implementation through the existing AosCloud/AosCore reference flow, then run the same scenarios on QEMU to verify behavioural parity.

This should be described as **Python prototyping followed by C++ productionization**, not as automatic Python-to-C++ conversion. The portable assets are the vehicle API contract, configuration, scenarios, expected behaviour, and tests; the C++ application remains a deliberate implementation that must be verified independently.

---

## Phase 1 Reference Implementation: QEMU

![Architecture Phase 1](./images/architecture_phase1.svg)

Phase 1 is the implemented reference environment. It runs the application and its supporting services across two QEMU-based Linux VMs, allowing the lifecycle, deployment, vehicle-data flow, and application behaviour to be exercised without automotive hardware.

### Component Distribution

| Layer | Current component | Responsibility |
|---|---|---|
| Hosted development environment | digital.auto Playground | Application editing and hosting, build/deployment trigger, runtime selection, and dashboard visualization |
| Lifecycle backend | AosCloud | Application registry, versioning, target configuration, and deployment orchestration |
| QEMU-VM-1 | AosCore, EV Range Extender application, KUKSA integration | Runs the main application and battery-related signal logic |
| QEMU-VM-2 | BMS, range, HVAC, and seat services | Simulates vehicle functions and the end-ECU domain |
| Cross-domain communication | Eclipse Zenoh | Exchanges data between the two virtual-machine domains |
| Vehicle data | Eclipse KUKSA and COVESA VSS | Provides a consistent vehicle-signal model and broker API |
| Host-side input | Hardware simulator | Generates the inputs used to validate the deployed system |

### System Setup Workflow

This section describes the end-to-end setup required to recreate the Phase 1 demo from scratch.

The setup is organized into sub-sections that guide the VM setup and deployment flow in a practical sequence.

- [Section 1 — VM setup and deployment flow](#section-1--vm-setup-and-deployment-flow)
- [Section 2 — Build and deploy the SDV application](#section-2--build-and-deploy-the-sdv-application)
- [Section 3 — AosEdge setup](#section-3--aosedge-setup)


#### Section 1 — VM setup and deployment flow

**Prepare the VM environment**

This step sets up two QEMU VM instances where the SDV application and its surrounding components are run.

- Download the latest Bosch Aos VM image package and provisioning script from the [meta-aos-vm releases](https://github.com/aosedge/meta-aos-vm/releases/) page. Select the latest image named `6.x.x-bosch.x`.

  - As an alternative to using the release images, you can build the VM image yourself by following [meta-aos-vm](https://github.com/aosedge/meta-aos-vm) on the `demo-bosch` branch.

- Extract the image archive and start the QEMU-based VMs from the same directory:

  ```bash
  tar -xvf aos-vm-image-genericx86-64-6.1.1-bosch.2.tar.xz
  sudo ./aos_vm.sh run -f .
  ```

- You may need to run `chmod +x aos_vm.sh` to allow the execution of `aos_vm.sh`.

- If the Aos certificates are unavailable or the setup has not been completed, follow the [Aos QuickStart](https://docs.aosedge.tech/docs/quick-start/).

  - Complete the QuickStart guide only through the **Get access** step. No additional QuickStart steps are required.
  - Perform these steps on WSL or Ubuntu.

  - Required steps:
    1. [Set up your host](https://docs.aosedge.tech/docs/quick-start/set-up/)
    2. [Get access](https://docs.aosedge.tech/docs/quick-start/get-access)

- Access the main node with `ssh root@10.0.0.100` and the secondary node with `ssh root@10.0.0.x`, where the address can be discovered with:

  ```bash
  ip neigh
  ```

- Use `Password1` as the password when prompted to log in to the main node and secondary node VMs.

- If needed, monitor the boot and service logs with `journalctl -f`.

- Provision the VMs to AosCloud on Host:

  ```bash
  source ~/.aos/venv/bin/activate
  aos-prov provision -u 10.0.0.100
  ```

- Log in to the [Aos Dashboard](https://api.aoscloud.io/account/start), select the OEM login option, and choose the certificate-based sign-in that appears when you open the [Units tab](https://oem.aoscloud.io/oem/units).

- If the unit appears offline in the Aos Dashboard, follow the [VM network troubleshooting steps](#vm-network).

**Install the core components**

This step installs the core components, including `kuksa-client`, `zenoh`, and `pylibs`.

- Download the Aos VM layers package from the same release page: [aos-vm layers package](https://github.com/aosedge/meta-aos-vm/releases/tag/v6.1.1-bosch.2)

- Extract the archive and publish the layers using the signing flow:

  ```bash
  source ~/.aos/venv/bin/activate
  tar -xvf aos-vm-layers-genericx86-64-6.1.1-bosch.2.tar.gz
  cd layers
  aos-signer go
  ```

- After the publish step, verify in the [AosCloud Layers-Service Provider](https://sp.aoscloud.io/sp/layers) portal that the expected layers are available in the Layers section. The layers that should appear are `kuksa-client`, `zenoh`, and `pylibs`.

- Check the [deployment bundles](https://oem.aoscloud.io/oem/deployment-bundles) to confirm that the layers were deployed successfully.

- If the deployment does not appear or is rejected, update the service version in `demo-services/ev-range-extender/config.yaml` and re-run `aos-signer go`.

**Deploy the demo services**

This step deploys the components that produce the data required for the SDV application (ev-range-extender).

- Before deploying the demo services, complete the [application-deployment troubleshooting step](#application-deployment) if name resolution between the VMs is not already configured.

- The `demo-services` folder contains the deployment bundles for the EV Range Extender use case: `bms`, `range-ai`, `seat-ecu`, and `hvac`.

- In the VM, navigate to the EV Range Extender service directory and package it for deployment:

  ```bash
  source ~/.aos/venv/bin/activate
  cd epam-service-connector/eclipse-sdv-blueprint/demo-services/ev-range-extender
  aos-signer go
  ```
- After the publish step, verify in the [AosCloud Service-Service Provider](https://sp.aoscloud.io/sp/services).

**Configure Playground dashboard connectivity**

- Deploy `kuksa-syncer`, which connects the running system to the Playground dashboard:

  ```bash
  source ~/.aos/venv/bin/activate
  cd epam-service-connector/eclipse-sdv-blueprint/kuksa-syncer
  aos-signer go
  ```

- Verify the deployment result in [Aos Dashboard Services](https://sp.aoscloud.io/sp/services).
- If the deployment does not appear or is rejected, update the service version in `kuksa-syncer/config.yaml` and re-run `aos-signer go`.
- Check the [deployment bundles](https://sp.aoscloud.io/sp/deployment-bundles) if an error occurs during deployment.

#### Section 2 — Build and deploy the SDV application

- Sign in to the digital.auto Playground at [playground.digital.auto](https://playground.digital.auto).
- Open the [EV Range Extender application](https://playground.digital.auto/model/67f76c0d8c609a0027662a69/library/prototype/69ce30f438bb8e98f0af5ac8/view).
- Select the [AosCloud Deployment plugin](https://playground.digital.auto/model/67f76c0d8c609a0027662a69/library/prototype/69ce30f438bb8e98f0af5ac8/plug?plugid=aos-cloud-deployment).
- Upload the Service Provider `.p12` certificate from `.aos/security`.
- In the AosCloud Deployment plugin, choose `C++`, select `EV Range Extender` from the dropdown menu, and click `Build and Deploy`.

#### Section 3 — AosEdge setup

The script `aos-automation.py` performs the end-to-end AosEdge setup, including unit-config updates, unit-set creation, subject creation, and service assignment. See [AosEdge setup (manual)](AosEdge%20setup%20(manual).md) for the equivalent manual procedure.

1. Change into the automation directory
   ```bash
   cd eclipse-sdv-blueprint/Aosedge-Automation
   ```

2. Create and activate the Python virtual environment
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   ```

3. Install the required Python dependencies
   ```bash
   pip install -r requirements.txt
   ```

4. Run the automation script
   ```bash
   python aos-automation.py
   ```
   It will prompt for the target unit system ID and then update the unit configuration automatically.

5. After deployment, log in to the units via SSH and verify that the services are running:

    ```bash
    crun --root=/run/crun list
    ```
    Service deployments can be verified on the [units portal](https://oem.aoscloud.io/oem/units) for the respective unit.

### Run the Demo

Complete Sections 1, 2, and 3 above before running the demo.

1. Open the [Playground dashboard](https://playground.digital.auto/model/67f76c0d8c609a0027662a69/library/prototype/69ce30f438bb8e98f0af5ac8/dashboard). In the right pane, click `Add Runtime`, enter `Runtime-ev-range-extender`, and click `Add`. Select `Runtime-ev-range-extender` from the runtime dropdown.

![EV Range Extender runtime selection on the playground dashboard](./images/image.png)

2. On the host, start the hardware simulator by running `./hardware-sim/pytk_hwsim.py` from the `eclipse-sdv-blueprint` directory. See [hardware-sim/README.md](hardware-sim/README.md) for details.

3. Click `Start` in the hardware simulator to begin the driving simulation.

**Observe the threshold-based behaviour:**

1. When the battery level reaches 50%, the HVAC fan is automatically turned off.
2. When the battery level reaches 30%, additional power-saving measures are applied, and the seat heating/cooling functions are turned off.
3. When the HVAC fan is turned off, a slight increase in the estimated driving range can be observed.
4. When the seat heating/cooling functions are also disabled, the estimated driving range increases further.
5. Log in to VM1 using SSH:

    ```bash
    ssh root@10.0.0.100
    ```

6. Monitor the application logs by running:

    ```bash
    journalctl -f | grep "range-ext"
    ```

7. Review the application-level logs and confirm that the expected threshold actions occurred.

### Signal Flow and Internals

The demo runs as a closed loop across the host, virtual machines, and Playground dashboard integration.

1. **Hardware simulator (host side)** publishes battery and cabin control values.
2. **VM-1 runtime stack** receives battery values and updates the vehicle signal broker.
3. **Vehicle-signal broker (KUKSA)** stores and distributes current vehicle values used by the application and runtime services.
4. **Bridge layer** transfers cabin-related signals and updates between QEMU-VM-1 and QEMU-VM-2 so that both compute domains remain synchronized.
5. **VM2 ECU services** apply HVAC and seat actions and publish actuator status to the dashboard integration.

### Troubleshooting

#### VM Network

Before performing these checks, verify the bridge and external interface names on your host and replace `aos-br0` / `eth0` if they differ.

1. Check that the bridge and IP forwarding are configured:

    ```bash
    ip addr show aos-br0
    cat /proc/sys/net/ipv4/ip_forward
    ```

2. The output should show `10.0.0.1/24` on the bridge and `1` for forwarding. If forwarding shows `0`, enable it:

    ```bash
    sudo sysctl -w net.ipv4.ip_forward=1
    ```

3. Check that a `MASQUERADE` rule exists:

    ```bash
    sudo iptables -t nat -L POSTROUTING -n -v
    ```

    If it's empty, re-add it using your external interface name instead of `eth0`:

   ```bash
   sudo iptables -t nat -A POSTROUTING -o <external-interface> -j MASQUERADE
   ```

4. Check that the `FORWARD` chain allows traffic in both directions:

    ```bash
    sudo iptables -L FORWARD -n -v
    ```
    It should show `aos-br0→<external-interface> ACCEPT` and `<external-interface>→aos-br0 ACCEPT` with state `RELATED,ESTABLISHED`. If missing:
    ```bash
    sudo iptables -A FORWARD -i aos-br0 -o <external-interface> -j ACCEPT
    sudo iptables -A FORWARD -i <external-interface> -o aos-br0 -m state --state RELATED,ESTABLISHED -j ACCEPT
    ```
    Use the actual interface name on your machine, for example `eth0`, `ens33`, `enp3s0`, or another host-facing NIC.

#### Application Deployment

1. SSH into the secondary VM:

    ```bash
    ssh ubuntu@10.0.0.X 
    ```
    If the filesystem is mounted read-only, remount it as writable:
    
    ```bash
    mount -o rw,remount /
    ```

    To find the secondary VM's IP address, run:

    ```bash
    ip neigh
    ```

2. Open `/etc/hosts` for editing with `vi`:

    ```bash
    vi /etc/hosts
    ```

3. Add the following entry to the file:

    ```text
    10.0.0.100 main
    ```

4. Save and exit `vi`:

   - Press `Esc`.
   - Type `:wq`.
   - Press `Enter`.

5. Verify DNS resolution for `main`:

    ```bash
    nslookup main
    ```


---

## Planned Phase 2: Physical Hardware

![Architecture Phase 2](./images/architecture_phase2.svg)

Phase 2 is a design target and is not yet available for trial. It is intended to replace the two virtual machines with representative HPC, zonal, and end-ECU hardware while preserving the VSS application contract and as much of the Phase 1 lifecycle as practical. The exact hardware integration and software distribution may change as implementation progresses.

The planned end-ECU layer uses an STM32-class microcontroller to represent the compute directly connected to sensors and actuators. The phase also introduces an explicit zonal layer between the HPC and end ECU.

### Planned Flow

```
1. Use the same reference lifecycle flow as Phase 1
        ↓
2. Run the application on an NXP S32G2-based HPC target
        ↓
3. Connect the HPC to a Raspberry Pi-based zonal target using Eclipse S-CORE / SOME/IP
        ↓
4. Connect the zonal target to an STM32-based end ECU using Eclipse Zenoh
        ↓
5. Exercise representative HVAC, display, and seat actuators
```

### Planned Component Distribution

| Layer | Candidate hardware | Planned software | Responsibility |
|---|---|---|---|
| Lifecycle backend | — | AosCloud reference backend | Registry and deployment orchestration, as in Phase 1 |
| HPC | NXP S32G2 | AosCore, EV Range Extender, and candidate Eclipse AutoSD integration | Runs the primary application logic |
| Zonal | Raspberry Pi | Linux and SDV integration services | Bridges the HPC and end-ECU domains |
| End ECU | STM32 | Eclipse ThreadX | Runs representative actuator control |
| HPC ↔ zonal | — | Eclipse S-CORE / SOME/IP | Provides the planned automotive middleware path |
| Zonal ↔ end ECU | — | Eclipse Zenoh | Provides lightweight pub/sub communication |

### Candidate Additional Signals

| Signal | Layer | Purpose |
|---|---|---|
| `Vehicle.Cabin.HVAC.TargetTemperature` | End ECU | Adjust climate control for power saving |
| `Vehicle.Infotainment.Display.Brightness` | End ECU | Dim screen to reduce power draw |
| `Vehicle.Cabin.Seat.Ventilation.Level` | End ECU | Disable seat ventilation |

These signals are candidates for expanding the physical-hardware demonstration; they are not implemented by the current Phase 1 setup.


### Additional Eclipse Projects Planned for Phase 2

| Component | Role |
|---|---|
| [Eclipse AutoSD](https://github.com/eclipse-autosd/eclipse-autosd) | Candidate automotive Linux integration for the HPC target |
| [Eclipse S-CORE](https://github.com/eclipse-score) | Planned middleware path between the HPC and zonal layers |
| [Eclipse Zenoh](https://github.com/eclipse-zenoh) | Planned zonal-to-end communication |
| [Eclipse ThreadX](https://github.com/eclipse-threadx/threadx) | Planned RTOS for the microcontroller target |

---

## Phase Comparison

| | Phase 1 | Phase 2 |
|---|---|---|
| **Status** | Implemented | Under development |
| **Goal** | Validate the deployed lifecycle and signal flow without automotive hardware | Exercise the same blueprint concepts on representative hardware |
| **HPC** | Linux VM (QEMU) | NXP S32G2 |
| **Zonal** | Not present | Raspberry Pi |
| **End ECU** | Linux VM (QEMU) | STM32 |
| **HPC ↔ End comms** | Eclipse Zenoh | — |
| **HPC ↔ Zonal comms** | — | Eclipse S-CORE / SOME/IP |
| **Zonal ↔ End comms** | — | Eclipse Zenoh |
| **Validation location** | Deployed QEMU environment | Planned physical targets |
| **Setup requirements** | Hosted Playground and AosCloud access, two QEMU VMs, and host-side simulator | Hardware, board support, and target-specific integration |

---

## Project Resources

| Resource | Link |
|---|---|
| digital.auto Playground | [playground.digital.auto](https://playground.digital.auto) |
| Development Repository | [eclipse-autowrx/epam-service-connector](https://github.com/eclipse-autowrx/epam-service-connector) |
| Eclipse SDV Blueprint Proposal | [eclipse-sdv-blueprints/blueprints#18](https://github.com/eclipse-sdv-blueprints/blueprints/issues/18) |
| Eclipse AutoWRX | [github.com/eclipse-autowrx](https://github.com/eclipse-autowrx) |
| AutoWRX SDV Runtime | [github.com/eclipse-autowrx/sdv-runtime](https://github.com/eclipse-autowrx/sdv-runtime) |
| Eclipse KUKSA | [github.com/eclipse-kuksa](https://github.com/eclipse-kuksa) |
| Eclipse Zenoh | [github.com/eclipse-zenoh](https://github.com/eclipse-zenoh) |
| Eclipse Velocitas | [github.com/eclipse-velocitas](https://github.com/eclipse-velocitas) |
| Velocitas Python SDK | [eclipse-velocitas/vehicle-app-python-sdk](https://github.com/eclipse-velocitas/vehicle-app-python-sdk) |
| Velocitas C++ SDK | [eclipse-velocitas/vehicle-app-cpp-sdk](https://github.com/eclipse-velocitas/vehicle-app-cpp-sdk) |
| digital.auto Website | [www.digital.auto](https://www.digital.auto) |
| AosEdge Source | [github.com/aosedge](https://github.com/aosedge) |
| AosEdge Documentation | [docs.aosedge.tech](https://docs.aosedge.tech/) |
| AosCloud | [AosCloud](https://api.aoscloud.io/account/start) |
