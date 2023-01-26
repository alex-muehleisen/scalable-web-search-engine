# Creating a Highly Scalable Web-search Engine

**Skills:**
1. Working with a moderately large software project
1. Parallelizing data analysis work
1. Work with WARC files and the multi-petabyte common crawl dataset
1. Indexes and rollup tables familiarity for speeding up queries
1. Debugging database performance problems

## Project Setup

1. Bring up all docker containers and verify everything works.
1. Running the `scripts/create_passwords.sh` script generates a `.env.prod` file containing production credentials for the database.
1. Enabling ssh port forwarding allows the local computer to connect to the running flask app.
1. Running the script:

   ```
   $ sh scripts/check_web_endpoints.sh
   ```
   
   performs automated integration checks to ensure the system is running correctly.
   
## Loading Data

1. Visit <https://commoncrawl.org/the-data/get-started/>
1. Find the url of a WARC file.
1. Run the command

   ```
   $ ./downloader_warc.sh $URL
   ```
   
   where `$URL` is the url to the selected WARC file.
   This command will spawn a docker container that downloads the WARC file and inserts it into the database.
1. These steps were repeated 5 times for 5 different WARC files, each from different years.

The `downloader_warc` service downloads many urls quickly but they are mostly low-quality urls.
For example, most URLs do not include the date they were published, so their contents will not be reflected in the ngrams graph.
The solution is to implement and run a `downloader_host` service for downloading high quality urls, which can be seen here in the `services/downloader_host/downloader_host.py` script.

## Speeding up the Webpage

Every time a web search on this website is made, several sql queries are run. Without an index to speed up the query, it will use a sequential scan and the runtime will be linear in the amount of data searched. This is quite burdensome. To solve this issue, the following RUM index was created to speed up the `@@` operator:

```
CREATE INDEX ON metahtml USING rum(content);
```
## Results 

This query shows the total number of webpages loaded:

```
select count(*) from metahtml;

count  
--------
231922
(1 row)
```
       
This query shows the number of webpages loaded / hour:
 ```
 select * from metahtml_rollup_insert order by insert_hour desc limit 100;

   hll_count |  url   | hostpathquery | hostpath |  host  |      insert_hour       
  -----------+--------+---------------+----------+--------+------------------------
           1 |  15902 |         15512 |    14814 |  10135 | 2022-05-06 20:00:00+00
           5 | 222697 |        208619 |   199718 | 174491 | 2022-05-06 19:00:00+00
  (2 rows)
 ```
       
This query shows the hostnames that had the most webpages downloaded from:
```
select * from metahtml_rollup_host order by hostpath desc limit 100;

url | hostpathquery | hostpath |             host             
-----+---------------+----------+------------------------------
106 |           106 |      106 | com,economist)
 61 |            61 |       61 | com,chamberofcommerce)
 49 |            49 |       49 | com,experts-exchange)
 46 |            46 |       46 | co,ello)
 44 |            44 |       44 | me,about)
 47 |            47 |       42 | gov,clinicaltrials)
 41 |            41 |       41 | edu,unm,econtent)
 41 |            41 |       41 | com,imdb)
 39 |            39 |       39 | tr,com,tripadvisor)
 38 |            38 |       38 | org,domestika)
 37 |            37 |       37 | com,ibegin)
 43 |            43 |       36 | org,mozilla,addons)
 35 |            35 |       35 | tv,veer)
 34 |            34 |       34 | com,aol)
 34 |            34 |       34 | ru,surfingbird)
 34 |            34 |       34 | my,com,tripadvisor)
 33 |            33 |       33 | org,w3,lists)
 33 |            33 |       33 | com,mihav)
 33 |            33 |       33 | com,smugmug,photos)
 33 |            33 |       33 | ua,in,read)
 32 |            32 |       32 | com,funnyjunk)
 32 |            32 |       32 | com,si)
 32 |            32 |       32 | net,mbc)
 31 |            31 |       31 | com,touristtube)
 31 |            31 |       31 | com,purelovers,work)
 30 |            30 |       30 | com,photobucket)
 30 |            30 |       30 | be,kuleuven,lirias)
 30 |            30 |       30 | su,eti)
 29 |            29 |       29 | com,archdaily,my)
 29 |            29 |       29 | com,wanderu)
 28 |            28 |       28 | org,debian,lists)
 28 |            28 |       28 | com,engadget)
 27 |            27 |       27 | dk,tripadvisor)
 27 |            27 |       27 | it,ebay)
 26 |            26 |       26 | com,foxnews,video)
 25 |            25 |       25 | uk,co,myindex)
 24 |            24 |       24 | com,merriam-webster)
 24 |            24 |       24 | org,le-guide-sante)
 24 |            24 |       24 | com,cargurus)
 24 |            24 |       24 | com,remax)
 24 |            24 |       24 | com,wallapop,es)
 24 |            24 |       24 | com,allposters)
 24 |            24 |       24 | ru,plati)
 24 |            24 |       24 | com,happycampus)
 24 |            24 |       24 | com,encycolorpedia)
 23 |            23 |       23 | com,pixels)
 23 |            23 |       23 | com,prezi)
 23 |            23 |       23 | ru,livemaster)
 23 |            23 |       23 | com,drewaltizer)
 23 |            23 |       23 | com,bendecho)
 23 |            23 |       23 | com,bibliatodo)
 23 |            23 |       23 | es,ebay)
 23 |            23 |       23 | com,chicagotribune,articles)
 23 |            23 |       23 | com,bandtoband)
 23 |            23 |       23 | ru,garant,base)
 23 |            23 |       23 | se,vakanser)
 23 |            23 |       23 | gov,loc,chroniclingamerica)
 23 |            23 |       23 | nl,onlinetouch)
 23 |            23 |       23 | com,hotellook)
 23 |            23 |       23 | com,ekitan)
 22 |            22 |       22 | com,xiachufang)
 22 |            22 |       22 | com,taptap)
 22 |            22 |       22 | com,thinkatheist)
 22 |            22 |       22 | com,leclubdesbonsvivants)
 22 |            22 |       22 | com,audiomack)
 22 |            22 |       22 | com,pinterest,id)
 22 |            22 |       22 | com,3d2f)
 22 |            22 |       22 | com,aftabir)
 22 |            22 |       22 | org,wikipedia,pl)
 22 |            22 |       22 | org,wikipedia,cs)
 29 |            29 |       21 | com,audible)
 21 |            21 |       21 | ru,a2ch)
 21 |            21 |       21 | org,wikipedia,en)
 21 |            21 |       21 | com,rottentomatoes)
 21 |            21 |       21 | com,google,patents)
 21 |            21 |       21 | org,wikipedia,nl)
 21 |            21 |       21 | com,oventrop)
 21 |            21 |       21 | com,yahoo,mall,tw)
 20 |            20 |       20 | ru,restoclub)
 20 |            20 |       20 | de,europages)
 20 |            20 |       20 | uk,co,telegraph)
 20 |            20 |       20 | org,opensubtitles)
 20 |            20 |       20 | com,chess)
 20 |            20 |       20 | com,apnews)
 20 |            20 |       20 | com,odnagazeta)
 20 |            20 |       20 | cc,limetorrents)
 19 |            19 |       19 | org,sae)
 19 |            19 |       19 | com,suicidegirls)
 19 |            19 |       19 | org,wikipedia,zh)
 19 |            19 |       19 | com,stylebistro)
 19 |            19 |       19 | com,familytreecircles)
 19 |            19 |       19 | com,books2read)
 19 |            19 |       19 | com,wilderssecurity)
 19 |            19 |       19 | uk,co,zoopla)
 20 |            20 |       19 | com,phoronix)
 19 |            19 |       19 | com,justjared)
 19 |            19 |       19 | lv,ss)
 19 |            19 |       19 | com,shababdz)
 19 |            19 |       19 | com,insideflyer)
 19 |            19 |       19 | org,wikipedia,ja)
(100 rows)
```
       
       
An example search result is included below. The search engine render time can be seen in the bottom-left.

<img src='search_engine_screenshot.png' />
