# OSRM-deployement

In this tuto, we will try to install OSRM by using docker

The project site can be found [here](http://project-osrm.org/)

The project git repo can be found [here](https://github.com/Project-OSRM)
The official installation doc can be found [here](https://github.com/Project-OSRM/osrm-backend)

## A quick install

### Step 1. Get the openstreet map

OSRM need to generate Map data to be able to calculate the shortest way between two locations. And it can generate these map data by using OpenStreetMap data in format [pbf](https://wiki.openstreetmap.org/wiki/PBF_Format).

You can a list of available maps in this [location](http://download.geofabrik.de/)

For example, we can download the map of `ile-de-france` with below command

```shell
# for linux
cd /home/pliu/osrm_data
wget http://download.geofabrik.de/europe/france/ile-de-france-latest.osm.pbf
```

```shell
# for windows
cd 'C:\Users\pengfei_account\Documents\osrm_data'
wget http://download.geofabrik.de/europe/france/ile-de-france-latest.osm.pbf
```

### Step 2. Transform openstreet map to OSRM format

The easiest way is to use the [osrm-backend docker image](https://hub.docker.com/r/osrm/osrm-backend/). This image contains the binary called `osrm-extract` which can do the data transformation.

Below commands are an example on how to use this docker image to the data transformation

```shell
# for linux and windows
docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-extract -p /opt/car.lua /data/ile-de-france-latest.osm.pbf
```

-t: this flag tells docker to allocate a virtual terminal session within the container. (If you want the terminal to be interactive, you can add -i option)

-v "${PWD}:/data": this flag tells the docker Daemon to start up a `volume` in the `target container` which uses `file system of the host`. In our case, docker will create a volume **/data** in the container which uses the current location (retrived from ${PWD}) of the host. It means the file in our host will be mounted to **/data/ile-de-france-latest.osm.pbf inside the container**.

Below are another two transformation that osrm need to transform the openstree map

```shell
# for linux and windows
docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-partition /data/ile-de-france-latest.osm.pbf
docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-customize /data/ile-de-france-latest.osm.pbf
```

After these two steps, you should see many files that start with `ile-de-france-latest.osm` and finish with various extension (e.g. .cell_metrics, edges, etc.).

### Step 3. Deploy the osrm-backend with generated data

```shell
docker run -t -i -p 5000:5000 -v "${PWD}:/data" osrm/osrm-backend osrm-routed --algorithm mld /data/ile-de-france-latest.osm
```

-p p1:p2 : The p2 is the port number of the application runs inside the container. The p1 is the port number that you want to expose to host. For example -p 80:8080, the tomcat runs with 8080 inside the container, we expose it to host with port 80(accessible via host web brower).

Note this time we run the docker image with command **osrm-routed**. It takes one option **--algorithm mld**, and the location of the map.

Note osrm support two algo:

- Contraction Hierarchies (ch)
- Multi-Level Dijkstra (mld)

we recommend using MLD by default except for special use-cases such as very large distance matrices where CH is still a better fit for the time being. If you want to use the `--algorithm ch`. You need to transform the map differently. You need to replace `osrm-partition` and `osrm-customize` with a single `osrm-contract` in step 2.

### Step 4. Test the backend

The backend offers a Rest api which answers request on port 5000 (by default). Below command is an example.

```shell
curl "http://127.0.0.1:5000/route/v1/driving/13.388860,52.517037;13.385983,52.496891?steps=true"
```

You can find the full doc of the API [here](https://github.com/Project-OSRM/osrm-backend/blob/master/docs/http.md)

### Step 5. Deploy the official frontend

To make the backend more user-friendly, OSRM provides an official frontend. You can find the docker image [here](https://hub.docker.com/r/osrm/osrm-frontend/).

Below command will run the frontend with port 9966

```shell
docker run -p 9966:9966 osrm/osrm-frontend
```
