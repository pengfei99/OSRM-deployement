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

#### OSRM Extract

Below commands are an example on how to use this docker image to the data transformation

```shell
# for linux and windows
docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-extract -p /opt/car.lua /data/ile-de-france-latest.osm.pbf
```

Docker command:

-t: this flag tells docker to allocate a virtual terminal session within the container. (If you want the terminal to be interactive, you can add -i option)

-v "${PWD}:/data": this flag tells the docker Daemon to start up a `volume` in the `target container` which uses `file system of the host`. In our case, docker will create a volume **/data** in the container which uses the current location (retrived from ${PWD}) of the host. It means the file in our host will be mounted to **/data/ile-de-france-latest.osm.pbf inside the container**.

OSRM command (osrm-extract):

-p /opt/car.lua : This option will set the profile for transformin the map from the .pbf into OSRM format by adding more information. For example autoroute speed limit.

You can find an example of the `car.lua` file [here](https://github.com/Project-OSRM/osrm-backend/blob/master/profiles/car.lua).

For more details on profiles, you can check this [page](https://github.com/Project-OSRM/osrm-backend/blob/master/docs/profiles.md)

#### OSRM partition and customize

Below are two other osrm transformation that osrm need to apply on the openstree map

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

docker command option:

-p p1:p2 : The p2 is the port number of the application runs inside the container. The p1 is the port number that you want to expose to host. For example -p 80:8080, the tomcat runs with 8080 inside the container, we expose it to host with port 80(accessible via host web brower).

OSRM command (osrm-routed)
The command **osrm-routed** takes one option **--algorithm mld**, and the location of the map data.

Note osrm support two algorithm:

- Contraction Hierarchies (ch)
- Multi-Level Dijkstra (mld)

we recommend using MLD by default except for special use-cases such as very large distance matrices where CH is still a better fit for the time being. If you want to use the `--algorithm ch`. You need to transform the map differently. You need to replace `osrm-partition` and `osrm-customize` with a single `osrm-contract` in step 2.

#### Change map data

If you want to change the map data, you need to follow the step 2 to curate the new map data. For example, if the new map data called `france-transfrontalier-2022.osrm.pbf`. You need to run below command to transform the data.

```shell
# note the map data must be compatible with the option file car.lua
docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-extract -p /opt/car.lua /data/france-transfrontalier-2022.osrm.pbf

docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-partition /data/france-transfrontalier-2022.osrm.pbf
docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-customize /data/france-transfrontalier-2022.osrm.pbf
 
```

After the transformation is done, you should see a file `france-transfrontalier-2022.osrm` is generated. You can run the backend with this newly generated ORSM map file

```shell
docker run -t -i -p 5000:5000 -v "${PWD}:/data" osrm/osrm-backend osrm-routed --algorithm mld /data/france-transfrontalier-2022.osrm
```

### Step 4. Test the backend

The backend offers a Rest api which answers request on port 5000 (by default). Below command is an example.

```shell
curl "http://127.0.0.1:5000/route/v1/driving/13.388860,52.517037;13.385983,52.496891?steps=true"
```

If everything works well, you should see below response

```text
{"code":"Ok","routes":[{"geometry":"mfp_I__vpArGo@dBdn@xtAmk@vRa@jLsBwAjSw@S","legs":[{"steps":[{"intersections":[{"out":0,"entry":[true],"bearings":[174],"location":[13.388798,52.517033]},{"out":2,"location":[13.388827,52.516848],"bearings":[0,90,180,270],"entry":[false,true,true,false],"in":0}],"driving_side":"right","geometry":"mfp_I__vpAB?^ENAJA|@If@E~AQLC","mode":"driving","duration":18.2,"maneuver":{"bearing_after":174,"location":[13.388798,52.517033],"bearing_before":0,"type":"depart"},"weight":20,"distance":154.8,"name":"Friedrichstra??e"},{"intersections":[{"out":3,"location":[13.389037,52.515649],"bearings":[0,90,180,270],"entry":[false,true,true,true],"in":0},{"out":3,"location":[13.386058,52.515451],"bearings":[0,90,180,270],"entry":[true,false,false,true],"in":1},{"out":2,"location":[13.384529,52.515349],"bearings":[90,165,270],"entry":[false,true,true],"in":0}],"driving_side":"right","geometry":"y}o_Io`vpA@Z@XF|BDfBPhG@H@b@@l@@D@f@LtE@f@@LH|DLhE@b@DlA?B?FDb@","mode":"driving","duration":91.6,"maneuver":{"bearing_after":264,"type":"turn","modifier":"right","bearing_before":172,"location":[13.389037,52.515649]},"weight":91.6,"distance":514.2,"name":"Behrenstra??e"},{"intersections":[{"out":1,"location":[13.381488,52.515141],"bearings":[75,165,255,345],"entry":[false,true,true,true],"in":0},{"lanes":[{"valid":false,"indications":["left"]},{"valid":true,"indications":["straight","right"]}],"out":1,"location":[13.382106,52.513965],"bearings":[75,165,255,345],"entry":[true,true,true,false],"in":3},{"out":1,"location":[13.383005,52.512253],"bearings":[75,165,255,345],"entry":[true,true,true,false],"in":3},{"out":2,"location":[13.383428,52.511468],"bearings":[30,105,165,240,345],"entry":[false,true,true,true,false],"in":4},{"out":1,"location":[13.384218,52.510007],"bearings":[90,165,270,345],"entry":[true,true,true,false],"in":3},{"out":1,"location":[13.385526,52.507427],"bearings":[90,165,270,345],"entry":[true,true,true,false],"in":3},{"out":1,"location":[13.386074,52.506433],"bearings":[90,165,345],"entry":[true,true,false],"in":2},{"out":0,"location":[13.386624,52.505366],"bearings":[165,255,345],"entry":[true,true,false],"in":2},{"out":1,"location":[13.387164,52.504286],"bearings":[90,165,345],"entry":[true,true,false],"in":2},{"out":1,"location":[13.387868,52.502938],"bearings":[60,165,240,345],"entry":[true,true,true,false],"in":3},{"out":1,"location":[13.388181,52.50234],"bearings":[75,165,345],"entry":[true,true,false],"in":2},{"out":2,"location":[13.388704,52.500711],"bearings":[0,75,180],"entry":[false,false,true],"in":0},{"out":2,"location":[13.388742,52.499423],"bearings":[0,120,180,285],"entry":[false,true,true,true],"in":0},{"out":2,"location":[13.388729,52.499301],"bearings":[0,60,180,270],"entry":[false,true,true,false],"in":0},{"out":2,"location":[13.388784,52.498126],"bearings":[0,105,165,285],"entry":[false,false,true,true],"in":0}],"driving_side":"right","geometry":"szo_IiqtpATIDC@Ad@SPGn@YfAe@DCBAPIRGFEv@[fBw@dCeATKzAo@\\ODCZOVKDCNGlD_Bh@UFCTKTIJEPIBAp@Y`Bq@FC^M\\Or@[^Oh@UrBy@PItDaBjAi@x@]FCdAa@r@YrB{@FAJEFCRIXMDCd@SJC`Ac@b@QVK^QDCVKVKDCRGf@U\\OFCn@WBAh@UTKFCVG\\G^E^EJ?PA??l@Ex@GLAnAIBBZ@j@BF?T@V@J?F?h@?nBEXAH?VATGHAv@ONC","mode":"driving","duration":218.2,"maneuver":{"bearing_after":161,"type":"turn","modifier":"left","bearing_before":261,"location":[13.381488,52.515141]},"weight":218.2,"distance":2025.9,"name":"Wilhelmstra??e"},{"name":"Mehringdamm","distance":169.7,"ref":"B 96","maneuver":{"bearing_after":168,"type":"new name","modifier":"straight","bearing_before":168,"location":[13.388931,52.497613]},"destinations":"A 100, B 96: Tempelhof","weight":21.5,"mode":"driving","geometry":"aml_Iy_vpAVGB?ZIFAnCg@r@MJCHAPC","intersections":[{"out":1,"location":[13.388931,52.497613],"bearings":[105,165,285,345],"entry":[true,true,false,false],"in":3}],"duration":21.5,"driving_side":"right"},{"intersections":[{"out":2,"location":[13.389349,52.496109],"bearings":[105,180,285,345],"entry":[false,true,true,false],"in":3},{"out":2,"location":[13.386905,52.496405],"bearings":[30,105,285],"entry":[true,false,true],"in":1}],"driving_side":"right","geometry":"ucl_ImbvpAEj@AFI`CGv@IfAQhCEf@Cd@IXMbB","mode":"driving","duration":32.5,"maneuver":{"bearing_after":278,"type":"turn","modifier":"right","bearing_before":171,"location":[13.389349,52.496109]},"weight":32.5,"distance":227,"name":"Obentrautstra??e"},{"intersections":[{"out":0,"location":[13.386085,52.496547],"bearings":[15,105,285],"entry":[true,false,true],"in":1},{"classes":["tunnel"],"out":0,"location":[13.386115,52.496632],"bearings":[15,195],"entry":[true,false],"in":1},{"out":0,"location":[13.38616,52.496756],"bearings":[15,195],"entry":[true,false],"in":1}],"driving_side":"right","geometry":"mfl_IanupAE?GCAAA?WGME","mode":"driving","duration":7.7,"maneuver":{"bearing_after":11,"type":"turn","modifier":"right","bearing_before":281,"location":[13.386085,52.496547]},"weight":7.7,"distance":31.8,"name":""},{"intersections":[{"in":0,"entry":[true],"bearings":[194],"location":[13.386189,52.496826]}],"driving_side":"right","geometry":"ehl_IunupA","mode":"driving","duration":0,"maneuver":{"bearing_after":0,"type":"arrive","modifier":"left","bearing_before":14,"location":[13.386189,52.496826]},"weight":0,"distance":0,"name":""}],"distance":3123.3,"duration":389.7,"summary":"Behrenstra??e, Wilhelmstra??e","weight":391.5}],"distance":3123.3,"duration":389.7,"weight_name":"routability","weight":391.5}],"waypoints":[{"hint":"FGMAgJUNBIAXAAAABQAAAAAAAAAgAAAAfXRPQdLNK0AAAAAAsPePQQsAAAADAAAAAAAAABAAAABIAQAA_kvMAKlYIQM8TMwArVghAwAA7wr5X-_H","distance":4.231666,"name":"Friedrichstra??e","location":[13.388798,52.517033]},{"hint":"UwcEgFYHBIATAAAAAAAAAAAAAAAAAAAAgY4AQQAAAAAAAAAAAAAAABMAAAAAAAAAAAAAAAAAAABIAQAAzUHMALoJIQP_QMwA-wkhAwAA7wD5X-_H","distance":15.756215,"name":"","location":[13.386189,52.496826]}]}
```
Or you can enter the query in your browser

`http://127.0.0.1:5000/route/v1/driving/2.29001,48.79969;2.42042,48.80516?alternatives=false&geometries=polyline&steps=false&overview=full&exclude=ferry`

You can find the full doc of the API [here](https://github.com/Project-OSRM/osrm-backend/blob/master/docs/http.md)

### Step 5. Deploy the official frontend

To make the backend more user-friendly, OSRM provides an official frontend. You can find the docker image [here](https://hub.docker.com/r/osrm/osrm-frontend/).

Below command will run the frontend with port 9966

```shell
docker run -p 9966:9966 osrm/osrm-frontend
```

If everything works well, open your browser and enter this url `http://127.0.0.1:9966/`, you should see below page.

![osrm_frontend.PNG](img/osrm_frontend.PNG)

### Step 6. Configure R package

There are two R package that can facilitate the use of OSRM backend server:

- metric.osrm
- metric-osrm-application-shiny

#### Install and configure metric.osrm

You can find the official git lab repo [here](https://git.lab.sspcloud.fr/metric-osrm/metric-osrm-package).

Below guide works for R>=3.4.0

```shell
# can be skipped, if its already installed
install.packages("remotes")

# install osrm metric
remotes::install_gitlab(repo = 'metric-osrm/metric-osrm-package',
                        host = 'git.lab.sspcloud.fr')
# import the package
library(metric.osrm)

# configure the package to connect to a osrm backend server 
options(osrm.server = "https://metric-osrm-backend.casd.eu/")

# setup profile type, it must be provided by the backend server, otherwise it won't work
options(osrm.profile = "driving") 
```

For more documentation about this package, you can visit this [page](https://metric-osrm.pages.lab.sspcloud.fr/metric-osrm-package/)

#### Install and configure metric-osrm-application-shiny

You can install a shiny app to use the osrm backend. For more information, please check this [page](https://git.lab.sspcloud.fr/metric-osrm/metric-osrm-application-shiny)

```R
# install the shiny app
remotes::install_gitlab(repo = 'metric-osrm/metric-osrm-application-shiny',
                        host = 'git.lab.sspcloud.fr')

# run the shiny app with a osrm backend server
shinyMetricOSRM::run_app(osrm.server = "https://metric-osrm-backend.casd.eu/")
```

If everthing works well, you should see a pop up window as shown in below figure

![osrm_shiny_app.PNG](img/osrm_shiny_app.PNG)

You can try to enter a starting a point and an ending point, can calculate the route.

If everthing works well, you can get the statistic of the route and the iteration on a map. Below figure
is an example

![osrm_rout_calc.PNG](img/osrm_rout_calc.PNG)


Note if you don't save your R studio work space after closing, when you start a new session of R studio, you need to reload the package and configure it

```shell
library(metric.osrm)

# configure the package to connect to a osrm backend server 
options(osrm.server = "https://metric-osrm-backend.casd.eu/")

# setup profile type, it must be provided by the backend server, otherwise it won't work
options(osrm.profile = "driving") 

shinyMetricOSRM::run_app(osrm.server = "https://metric-osrm-backend.casd.eu/")
```

If you saved your R studio work space, you can just run the shiny app directly
