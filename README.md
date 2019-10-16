# geoclimate

Geoclimate chain is a collection of spatial processes to produce vector maps of cities. First, algorithms are used to extract and transform OpenStreetMap data to a set of GIS layers (GeoClimate abstract model). The GIS layers are then processed  to compute urban indicators at three geographic scales : RSU, block and building (Bocher et al, 2018). These indicators are used to feed the TEB model, classify the urban tissues and build the LCZ zones.

The algorithms of the Geoclimate chain are implemented on top of the open source library OrbisData (https://github.com/orbisgis/orbisdata). Orbisdata provides a unique access point to query, manage, retrieve data from a PostGIS or a H2GIS database. Orbisdata is based on lambda expressions and sugar programming methods introduced since JAVA 8. Orbisdata is closed to Groovy syntax and provide an elegant and fluent framework to easily manage geospatial data. The processing chain is packaged in the GeoClimate repository available as a set of  groovy scripts.

# Download

Geoclimate is avalaible as maven artifact from the repository http://nexus.orbisgis.org

To use the current snapshot add in the pom

```xml
<dependency>
  <groupId>org.orbisgis</groupId>
  <artifactId>geoclimate</artifactId>
  <version>1.0-SNAPSHOT</version>
  <type>pom</type>
</dependency>
```


# How to use

The simple way to use the Geoclimate chain is to run it in a Groovy console, using Grab annotation (http://groovy-lang.org/groovyconsole.html).

Put the following script and run it to extract OSM data from a place name and transform it to a set of GIS Layers.

```groovy
// Declaration of the maven repository
@GrabResolver(name='orbisgis', root='http://nexus-ng.orbisgis.org/repository/orbisgis/')
@Grab(group='org.orbisgis', module='prepare_data', version='1.0-SNAPSHOT')

import org.orbisgis.datamanager.h2gis.H2GIS
import org.orbisgis.PrepareData
import org.orbisgis.processmanagerapi.IProcess

//Create a local H2GIS database
def h2GIS = H2GIS.open('/tmp/osmdb;AUTO_SERVER=TRUE')

//Run the process to extract OSM data from a place name and transform it to a set of GIS layers needed by Geoclimate
IProcess process = PrepareData.OSMGISLayers.extractAndCreateGISLayers()
         process.execute([
                datasource : h2GIS,
                placeName: "Vannes"])
 
 //Save the GIS layers in a shapeFile        
 process.getResults().each {it ->
        if(it.value!=null && !it.value.isInteger()){
                h2GIS.getTable(it.value).save("/tmp/${it.value}.shp")
            }
        }

```
The next script computes all geoindicators needed by the TEB model (http://www.umr-cnrm.fr/spip.php?article199). To run it the user must set a place name and a connexion to a spatial database (H2GIS or PostGIS). As described above, the script extract the OSM data and transform it to a set of GIS layers requiered by the Geoclimate chain. Then a set of algorithms are excecuted to compute, the 3 geounits (building, block and RSU). For each geounits geographical parameters like density, form, compacity, distance, sky view factor are computed.

```groovy
@GrabResolver(name='orbisgis', root='http://nexus-ng.orbisgis.org/repository/orbisgis/')
@Grab(group='org.orbisgis', module='processing_chain', version='1.0-SNAPSHOT')


import org.orbisgis.datamanager.h2gis.H2GIS
import org.orbisgis.processmanagerapi.IProcess
import org.orbisgis.processingchain.ProcessingChain


        String directory ="/tmp/geoclimate_chain"
        File dirFile = new File(directory)
        dirFile.delete()
        dirFile.mkdir()
        H2GIS datasource = H2GIS.open(dirFile.absolutePath+File.separator+"geoclimate_chain_db;AUTO_SERVER=TRUE")
        IProcess process = ProcessingChain.GeoclimateChain.OSMGeoIndicators()
        if(process.execute(datasource: datasource, placeName: "Cliscouet,Vannes")){
            IProcess saveTables = ProcessingChain.DataUtils.saveTablesAsFiles()
            saveTables.execute( [inputTableNames: process.getResults().values()
                                 , directory: directory, datasource: datasource])
        }
```

# Use Geoclimate in DBeaver

DBeaver is an opensource multi-platform database tool to query, explore, manage data (https://dbeaver.io/). Since the  6.2.2 version, DBeaver support the H2GIS-H2 database. User is able to create an H2GIS database, query and display spatial objects from a friendly interface (see https://twitter.com/H2GIS/status/1181566934548176897).

To use Geoclimate scripts in DBeaver, user must install the Groovy Editor developed by the OrbisGIS team.  

In DBeaver, go to 

    1. Main menu Help -> Install New Software
    2. Paste extension P2 repository URL http://devs.orbisgis.org/eclipse-repo into Work with field,
    press Enter
    3. Select Groovy Editor item
    4. Click Next->Finish. Restart DBeaver.

Once DBeaver has restarted, select the main menu Groovy Editor, clic on Open editor, then you will have a Groovy Console.
Copy-paste the previous script to use Geoclimate.



### Notes

Geoclimate uses a spatial database to store and process in a SQL way the data. The default datase is H2GIS but the user can set a PostGIS connection.

 - The temporary tables should respect the pattern : `tableName_UUID` with `-` replaced by `_` if needed.
 - Index should be create using the Postgresql syntax : `CREATE INDEX IF EXISTS indexName ON table(columnName) USING RTREE`.
 - The processes should be documented with a description of the process followed by the inputs with `@param` and then the outputs with `@return`. As example :
    ``` java
    /**
    * Description of my process
    * @param inputName Input description
    * @param inputName Input description
    * @return outputName Output description
    */
    ```
