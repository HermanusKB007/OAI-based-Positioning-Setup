🚀 # OAI-based-Positioning-Setup
This repository will accompany the research paper: A Framework for UL-TDoA-based 5G Positioning Experimentation using OAI.

This is an adaption and improvement build on the work in: [OAI-NRPPA-PROCEDURES](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/NRPPA_Procedures/doc/tutorial_resources/Positioning%20Tutorial/Positioning%20Testing%20Setup%20Guide.docx?ref_type=heads)

## Some helpful links to take note off
1. [NR_SA_Tutorial_OAI_CN5G](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/NR_SA_Tutorial_OAI_CN5G.md#22-oai-cn5g-configuration-files)
2. [NR_SA_Tutorial_OAI_nrUE](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/NR_SA_Tutorial_OAI_nrUE.md#65-lower-latency-on-user-plane)
3. [RF Simulator notes](https://gitlab.eurecom.fr/oai/openairinterface5g/-/tree/develop/radio/rfsimulator)
4. [NR SA OAI CN5G](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/NR_SA_Tutorial_OAI_CN5G.md)
5. [Building Scopes](https://gitlab.eurecom.fr/oai/trainings/oai-workshops/-/tree/main/ran?ref_type=heads#scopes)
6. [Channel Modelling](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/openair1/SIMULATION/TOOLS/DOC/channel_simulation.md)
7. [MAC](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/MAC/mac-usage.md)
8. [More MAC](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/RUNMODEM.md)

## Part 1: Single gNB

### Step 1 Pull Core Network Functions
```
docker pull oaisoftwarealliance/ims:latest
docker pull oaisoftwarealliance/oai-amf:develop
docker pull oaisoftwarealliance/oai-nrf:develop
docker pull oaisoftwarealliance/oai-smf:develop
docker pull oaisoftwarealliance/oai-udr:develop
docker pull oaisoftwarealliance/oai-upf:develop
docker pull oaisoftwarealliance/oai-udm:develop
docker pull oaisoftwarealliance/oai-ausf:develop
docker pull oaisoftwarealliance/oai-lmf:develop
docker pull oaisoftwarealliance/trf-gen-cn5g:latest
```
### Step 2 Pull RAN Repository and Bulid gNB and nrUE

Create working folder for Example OAI_RAN

```
cd OAI_RAN
git clone https://gitlab.eurecom.fr/oai/openairinterface5g.git 
cd openairinterface5g/cmake_targets
git checkout NRPPA_Procedures / 5ba375c7122dd4d1aabb1ff1d8772cc938201b2d
./build_oai -I # install dependencies
./build_oai --gNB --nrUE --ninja --build-lib telnetsrv -w SIMU -C
```

### Step 3 Start the 5G core

```
cd OAI_RAN/openairinterface5g/doc/tutorial_resources/oai-cn5g
sudo docker compose up -d
```

Check the status of the core if all NF are healthy 
`docker ps -a`

To stop the core
`sudo docker compose -f docker-compose.yaml down`

### Step 4 Starting gnb

Open a terminal 
```
cd OAI_RAN/openairinterface5g/cmake_targets/ran_build/build 
sudo ./nr-softmodem --rfsim --rfsimulator.serveraddr server --sa -O [path/to/conf/2x2.conf] --rfsimulator.options chanmod --gNB.[0].min_rxtxtime 6 --telnetsrv --T_stdout 2
```
Make sure to have channelmod conf file in conf folder for rfsimulator testing


#### Create a new file called "ue.sa.conf" and add
```
sa=1;
rfsim=1;

uicc0 = {
  imsi = "001010000000001";
  key = "fec86ba6eb707ed08905757b1bb44b8f";
  opc= "C42449363BBAD02B66D16BC975D77CC1";
  dnn= "oai";
  nssai_sst=1;
}

telnetsrv = {
  listenport = 9091
  histfile = "~/history.telnetsrv"
}

rfsimulator :
{
    serveraddr = "127.0.0.1";
    serverport = "4043";
    options = (); #("saviq"); or/and "chanmod"
    modelname = "AWGN"
};

@include "channelmod_rfsimu.conf"
```
### Step 6 Starting UE 
Please note: for extra commands use 
```
./nr-uesoftmoden --help
```
Open a terminal 
```
cd OAI_RAN/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3619200000 --rfsim --rfsimulator.serveraddr 127.0.0.1 --sa -O [path/to/conf] --rfsimulator.options chanmod --ue-nb-ant-rx 2 --ue-nb-ant-tx 2 --telnetsrv --telnetsrv.listenport 9090
```

### Step 6 External API to initiate Positioning request

Send the http post request at LMF address (as per our docker compose file http://192.168.70.141:80) to LMF determine location API to initiate the Positioning procedure

```
cd OAI_RAN/doc/tutorial_resources/positioning_tutorial/conf
curl --http2-prior-knowledge -H "Content-Type: application/json" -d "@InputData.json" -X POST http://192.168.70.141:8080/nlmf-loc/v1/determine-location
```

### Step 7: Collect Channel Estimate Measurements
At GNB side, run the ./nr-softmodem with tis extra command at the end:
```
--T_stdout 2
```

Then, open a new terminal window:
```
cd ../ran_build/build/common/utils/T/tracer
./record -d ../T_messages.txt -o ~/Desktop/channel_estimates.raw -on GNB_PHY_UL_TIME_CHANNEL_ESTIMATE
./extract -d ../T_messages.txt ~/Desktop/... GNB_PHY_UL_TIME_CHANNEL_ESTIMATE chest_t -o ~/Desktop/data.raw -count 100000
```

MATLAB script to process the extracted data:
```
% Simple SRS data analysis script
% Based on OAI positioning measurement data

% Set FFT size (adjust as needed)
Nfft = 2048 / 4096;

% Open the file
fileID = fopen('extracted.raw');

% Read int16 IQ data
IQ = fread(fileID,'int16');

% Separate I and Q components
I = IQ(1:2:end); % separate I
Q = IQ(2:2:end); % separate Q

% Form a complex signal
IQ = I + 1j*Q;

% Reshape into FFT-sized chunks
impulse_response = reshape(IQ, Nfft, []);

% Close file
fclose(fileID);
```

## Part 2: Multi gNB (on same host)

### Deploy 5G core
### Setup IP addresses for gNBs

For the 2nd and 3rd gNB we need to add IP addresses. For convenience we add them directly to the docker network that has been created for the core network. 
```
sudo ip address add 192.168.70.142/26 dev oai-cn5g
sudo ip address add 192.168.70.143/26 dev oai-cn5g
```
### Start UE

In the multi-gNB setup we need to invert the roles of client and server for the rfsimulator to make sure the gNBs are synchronized among themselves. I.e. the UE becomes the rfsimulator server and the gNBs the rfsimulator clients. 

```
 sudo ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3619200000 --rfsim --sa --rfsimulator.serveraddr server --uicc0.imsi 001010000000001
```

### Start gNBs

Once the core is up and IPs are set we can 

```
sudo ./nr-softmodem --rfsim --rfsimulator.serveraddr 127.0.0.1 --sa -O ~/OAI_RAN/doc/tutorial_resources/positioning_tutorial/conf/gnb1.sa.band78.106prb.rfsim4x4.conf
sudo ./nr-softmodem --rfsim --rfsimulator.serveraddr 127.0.0.1 --sa -O ~/OAI_RAN/doc/tutorial_resources/positioning_tutorial/conf/gnb2.sa.band78.106prb.rfsim2x2.conf
sudo ./nr-softmodem --rfsim --rfsimulator.serveraddr 127.0.0.1 --sa -O ~/OAI_RAN/doc/tutorial_resources/positioning_tutorial/conf/gnb3.sa.band78.106prb.rfsim.conf
```

Here in this example, just to check multiple TRPs is working fine, we are running gNB 1 with 4 TRPs, gNB 2 with 2 TRPs, and gNB3 with 1 TRP. So in our measurement response, each gNB should be sending ToA as per their TRP number.


### Initiate the localization procedure

```
curl --http2-prior-knowledge -H "Content-Type: application/json" -d "@InputData.json" -X POST http://192.168.70.141:8080/nlmf-loc/v1/determine-location
```

