# Intel eHSM Deployment Guide for Virtual Machine Environments
This document demonstrates how to deploy Intel eHSM in a virtual machine (VM).
Currently, Intel has discontinued official support for Intel eHSM, but its source code remains archived and publicly accessible.

However, Intel eHSM is designed to work with remote attestation, which requires access to Intel PCS (Provisioning Certification Service).
After reviewing several documents on Intel's remote attestation architecture, I decided to write this documentation to explain the integration process between remote attestation and Intel eHSM.

This document is also intended for those who prefer deploying the service on a virtual machine (VM) rather than directly on the host.
As far as I know, remote attestation is supported in virtualized environments, but I was unable to register platform within a VM.

In this setup, the host system is responsible for generating the PCKID and registering the platform ID.
All installation steps have been tested on `Ubuntu 24.04 LTS: Noble Numbat` with the Linux kernel version `6.8.0-58-generic`. 

Installation Verified On: 2025-04-25

## Host Environment Setup (Bare-Metal Required)

⚠️ **Important** 

The steps in this section **must be performed on a physical (bare-metal) host**, not within a virtual machine or container.
 Intel's Platform Registration service is only available on a subset of server-grade CPUs that support SGX.
If the sixth field of the PCKID -`platform manifest`- cannot be properly retrieved during the PCKID retrieval process, the machine is not eligible for attestation and therefore cannot be used with Intel eHSM. This process needs to be done **only once on the host**, after which guest virtual machines can freely perform SGX attestation.

### 0. Register SGX Repo Key
Intel SGX repository key for package access:

```bash
wget https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key 
cat intel-sgx-deb.key | sudo tee /etc/apt/keyrings/intel
sgx-keyring.asc > /dev/null
```

### 1. Install Intel PSW and PCK ID retrieval tool

```bash
sudo apt-get update
sudo apt install -y libsgx-psw
sudo apt install -y sgx-pck-id-retrieval-tool
```

### 2. Run PCK ID Retrieval Tool
 Fetch and register your SGX platform's PCK ID using PCCS or Intel PCS:
(You may need `root` or `sgx_prv` privilege to run this)
```bash
PCKIDRetrievalTool -f host_$(hostnamectl --static).csv -use_secure_cert false
```
- If a PCCS server is present, registration is automatic.
- If not, manually register the platform_manifest info to your Intel PCS server.

```bash
sudo apt-get install -y csvtool
bash -c "csvtool col 6 host_$(hostnamectl --static).csv | xxd -r -p > host_$(hostnamectl --static)_pm.bin"
curl -H "Content-Type: application/octet-stream" -v --data-binary @host_$(hostnamectl --static)_pm.bin -X POST "https://api.trustedservices.intel.com/sgx/registration/v1/platform"
```

If the registration is successful, the IRS will return a "HTTP/1.1 201 Created" reply, with the PPID of the registered platform as content. 
Sample response:

```
HTTP/1.1 201 Created
Content-Length: 32
Content-Type: text/plain
Request-ID: <request id>
Date: <date>
<PPID>
```

---

## Guest (Application) Setup

* I will skip the installation of SGX dependencies on the guest machine. For detailed instructions, please refer to [Intel's SGX repository](https://github.com/intel/linux-sgx).

### 0. Intel API Key
Login to the [Intel Developer Portal](https://api.portal.trustedservices.intel.com/) and generate your API key.

### 1. Clone ehsm repo and Update Environment Variables

```bash
git clone https://github.com/intel/ehsm
cd ehsm
```

Edit the following environment files:

- `docker/.env`
- `docker/.pccs.env`

Be sure to set `API_KEY` and `PCCS_URL (ex. PCCS_URL=https://your_ip:8081)` in `.pccs.env` You may change other secrets in these configs.


### 2. Build Docker Images

```bash
cd docker
docker compose build
```

### 3. Start PCCS Only (for setup and debugging)

```bash
docker compose up pccs -d
 ```

Check logs and ensure PCCS is listening and issuing certs.

```bash
docker compose logs pccs
```

### 4. Start the other Services in Background

```bash
docker compose up -d
```

Check the service log
```bash
docker compose logs [service-name]
```

To ensure proper SGX certification flow, start containers in this order:

1. `pccs`
2. `dkeyserver`
3. `dkeycache`
4. `ehsmKMS`

---

## Testing the Setup

After the containers are running, you can test the functionality of eHSM by executing the test binary inside the `ehsmKMS` container:
You can also test other apis in the [document](https://github.com/intel/ehsm/blob/main/docs/API_Reference.md).

```bash
docker exec -it <ehsm-kms-container-name> /home/ehsm/out/ehsm-core/ehsm_core_test
```

> Replace `<ehsm-kms-container-name>` with the actual name of the running container for `ehsmKMS` image. Default name is `c_ehsmKMS`

If the test completes successfully, you should see the following log message:
```text
INFO [Test/function_test.cpp: line 1776] - All of tests done. 61/61 success
```
This confirms that all tests have passed and eHSM is functioning correctly.


---

## Resources
- [Intel eHSM GitHub Repo](https://github.com/intel/ehsm/)
- [Intel SGX Downloads](https://download.01.org/intel-sgx/)
- [Intel PCCS API Documentation](https://api.portal.trustedservices.intel.com/)
- [Intel Platform Registration Guide](https://cc-enabling.trustedservices.intel.com/intel-tdx-enabling-guide/02/infrastructure_setup/#platform-registration)
