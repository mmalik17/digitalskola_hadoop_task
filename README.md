# HOMEWORK WEEK 5 
## ETL PROCESS FROM POKEMON WEBSITE TO HADOOP ENVIRONMENT
<b> PART A: FILE CREATION </b>
1. Create folder to store all the files needed. In this homework, the folder is named as ‘hadoop_pokeapi’ 
   <br>
2. Create a file called 'docker-compose.yml' inside ‘hadoop_pokeapi’ folder. Fill the file with Docker language, as follows: <br>
version: '3.8'

services:
  namenode:
    image: bde2020/hadoop-namenode:2.0.0-hadoop2.7.4-java8
    container_name: namenode
    platform: linux/amd64
    environment:
      - CLUSTER_NAME=test
      - CORE_CONF_fs_defaultFS=hdfs://namenode:8020
      - HDFS_CONF_dfs_replication=1
    ports:
      - "9870:9870"
      - "9000:9000"
    volumes:
      - namenode_data:/hadoop/dfs/name
      - shared_data:/shared
    networks:
      - hadoop

  datanode:
    image: bde2020/hadoop-datanode:2.0.0-hadoop2.7.4-java8
    container_name: datanode
    platform: linux/amd64
    environment:
      - CORE_CONF_fs_defaultFS=hdfs://namenode:8020
    volumes:
      - datanode_data:/hadoop/dfs/data
    networks:
      - hadoop
    depends_on:
      - namenode

  python:
    build:
      context: .
      dockerfile: Dockerfile.python
    container_name: python_container
    volumes:
      - shared_data:/app/output
    networks:
      - hadoop
    depends_on:
      - namenode

volumes:
  namenode_data:
  datanode_data:
  shared_data:

networks:
  hadoop:

3. Inside the folder, create a python image that run a script to store pokebase api to csv file. The image is saved with py file format. In this task, the file is named as hit_pokeapi.py
Write the python code as follows: <br>
import requests
import csv

def fetch_ability_details(ability_id):
    url = f"https://pokeapi.co/api/v2/ability/{ability_id}"
    response = requests.get(url)
    if response.status_code == 200:
        return response.json()
    else:
        return None

def save_to_csv(abilities, filename):
    with open(filename, 'w', newline='', encoding='utf-8') as csvfile:
        fieldnames = ['id', 'pokemon_ability_id', 'effect', 'language', 'short_effect']
        writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
        writer.writeheader()
        for ability in abilities:
            writer.writerow(ability)

def main():
    abilities = []
    for i in range(1, 1000):
        ability_data = fetch_ability_details(i)
        if ability_data:
            effect_entries = ability_data.get('effect_entries', [])
            if effect_entries:
                abilities.append({
                    'id': i,
                    'pokemon_ability_id': ability_data['id'],
                    'effect': effect_entries[0].get('effect', ''),
                    'language': effect_entries[0].get('language', ''),
                    'short_effect': effect_entries[0].get('short_effect', '')
                })
        if i % 100 == 0:
            save_to_csv(abilities, f'result_{i-99}_{i}.csv')
            abilities = []
if __name__ == "__main__":
    main()

4. Create txt file that store the library required to install. In this case, the library required is 'requests'
<br>
5. Create a Dockerfile.python to runs a python container in Docker. Write the code as follows: <br>
FROM python:3.9

# Create a directory to store the script and CSV files
WORKDIR /app/output/
COPY requirements.txt /app/output/
RUN pip install --no-cache-dir -r requirements.txt
# Copy the Python script into the container
COPY hit_pokeapi.py /app/output/
# Run the Python script to generate CSV files
CMD ["python3", "hit_pokeapi.py"]
 <br>
6. Create a file called config. The config code is from the link: https://hub.docker.com/r/apache/hadoop <br>
Write the code as follows:
CORE-SITE.XML_fs.default.name=hdfs://namenode
CORE-SITE.XML_fs.defaultFS=hdfs://namenode
HDFS-SITE.XML_dfs.namenode.rpc-address=namenode:8020
HDFS-SITE.XML_dfs.replication=1
MAPRED-SITE.XML_mapreduce.framework.name=yarn
MAPRED-SITE.XML_yarn.app.mapreduce.am.env=HADOOP_MAPRED_HOME=$HADOOP_HOME
MAPRED-SITE.XML_mapreduce.map.env=HADOOP_MAPRED_HOME=$HADOOP_HOME
MAPRED-SITE.XML_mapreduce.reduce.env=HADOOP_MAPRED_HOME=$HADOOP_HOME
YARN-SITE.XML_yarn.resourcemanager.hostname=resourcemanager
YARN-SITE.XML_yarn.nodemanager.pmem-check-enabled=false
YARN-SITE.XML_yarn.nodemanager.delete.debug-delay-sec=600
YARN-SITE.XML_yarn.nodemanager.vmem-check-enabled=false
YARN-SITE.XML_yarn.nodemanager.aux-services=mapreduce_shuffle
CAPACITY-SCHEDULER.XML_yarn.scheduler.capacity.maximum-applications=10000
CAPACITY-SCHEDULER.XML_yarn.scheduler.capacity.maximum-am-resource-percent=0.1
CAPACITY-SCHEDULER.XML_yarn.scheduler.capacity.resource-calculator=org.apache.hadoop.yarn.util.resource.DefaultResourceCalculator
CAPACITY-SCHEDULER.XML_yarn.scheduler.capacity.root.queues=default
CAPACITY-SCHEDULER.XML_yarn.scheduler.capacity.root.default.capacity=100
CAPACITY-SCHEDULER.XML_yarn.scheduler.capacity.root.default.user-limit-factor=1
CAPACITY-SCHEDULER.XML_yarn.scheduler.capacity.root.default.maximum-capacity=100
CAPACITY-SCHEDULER.XML_yarn.scheduler.capacity.root.default.state=RUNNING
CAPACITY-SCHEDULER.XML_yarn.scheduler.capacity.root.default.acl_submit_applications=*
CAPACITY-SCHEDULER.XML_yarn.scheduler.capacity.root.default.acl_administer_queue=*
CAPACITY-SCHEDULER.XML_yarn.scheduler.capacity.node-locality-delay=40
CAPACITY-SCHEDULER.XML_yarn.scheduler.capacity.queue-mappings=
CAPACITY-SCHEDULER.XML_yarn.scheduler.capacity.queue-mappings-override.enable=false

PART B: RUN THE FILE WITH TERMINAL
1. Open the terminal in VS Code. The terminal could be accesed by clicking view in menu bar, then click Terminal
2. Go to the ‘hadoop_pokeapi’ directory using ‘cd’ command
3. Run the following command line:
    a. docker-compose up
       This code will run the Dockerfile.python, which would generate all csv files and store it in the docker directory
       To check whether the csv files is generated or not, use ‘ls’ command
     b. docker exec -it python_container /bin/bash
       This code will run a namenode container
     c. docker-compose exec namenode hadoop fs -mkdir -p /path/in/hdfs/
       This code will create a folder in hadoop environment in namenode container.
     d. docker exec namenode bash -c 'for file in /shared/*.csv; do hadoop fs -put "$file" /path/in/hdfs/; done'
        This code is used to copy all csv files from namenode container in Docker, to the hadoop   environment.
     e. docker-compose exec namenode hadoop fs -ls /path/in/hdfs/
		    This code will check whether the files are uploaded or not


