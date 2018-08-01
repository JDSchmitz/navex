# navex
Navex is an exploit generation framework for web applications. It is composed of two main steps: (1) vulnerable sinks identification by performing static analysis, and (2) the generation of concrete exploits through dynamic analysis of web apps, for the identified vulnerable sinks. Navex extends/uses many open-source tools: Joern, PHPJoern, Z3, Z3-str2, crawler4j, Narcissus JavaScript Engin, and Xdebug. For more information on Navex, please read our paper "Precise and Scalable Exploit Generation for Dynamic Web Applications" published at USENIX Security 2018.

# Step 1: vulnerable sinks identification #
For Step 1, we enhanced Joern and PHPJoern. The enhanced tools are forks of the original Joern and PHPJoern, and available at https://github.com/aalhuz/joern/tree/navex and https://github.com/aalhuz/phpjoern/tree/navex.   


## Using our PHPJoern and Joern forks ##

- Follow all installation instructions at https://github.com/aalhuz/phpjoern/tree/navex.
- Before parsing an application using PHPJoern, the database schema of the application has to analyzed and formatted as a CSV file.
The dbAnalysis package in https://github.com/aalhuz/joern/tree/navex/projects/extensions/joern-php/src/main/java/dbAnalysis will parse the schema files and produce one file (by default called schema.csv) that has the schema information as CSV file.

- Run the main class in DBAnalysis.java and provide the directory that has the schema files. For example

			cd joern/projects/extensions/joern-php
			java -classpath "build/libs/*:lib/*" dbAnalysis/DBAnalysis


- TO run the parser, you have to supply the database schema file (i.e., schema.csv) as the following example
  			
        ./php2ast -f jexp -n nodes.csv -r edges.csv -d $PATH/schema.csv $APPLICATION

$PATH is the path to the schema.csv file, and $APPLICATION is the application to parse.

- Edit joern/projects/extensions/joern-php/build.gradle as explained in the file. 

- Follow the rest of the instructions on how to generate code property graphs with Joern and import them into Neo4j.

## Graph Traversals guided by our Attack Dictionary ##

To find vulnerabilities using our attack dictionary, we need to search the enhanced Code Property graph using gremlin queries (graph traversals). We have added several Joern-steps in our python-joern fork at https://github.com/aalhuz/python-joern/tree/navex.
 
- Follow the installation instructions at https://github.com/aalhuz/python-joern/tree/navex. The python wrapper static-main.py is the script that invokes Analysis.py, which has our attack dictionary.

- The traversals output will be in results/static_analysis_results.txt and results/include_map_results.txt. The first file has the analysis results that summarizes all found vulnerable paths and safe sinks as well. The vulnerable paths are written as TAC formulas as described in the paper. The Second file has PHP files inclusion relationships, which is going to be used in Step 2. Note, the paths to the result files are hardcoded in static-main.py and need to be changed before running the python script.

## Generating exploit strings (exploit seeds) using Z3 solver ##
Prerequisites: install Z3 solver and Z3-str2 extension. We have used Z3-str2 in Navex's implementation (not Z3-str3 which was not available during our evaluation). You can find Z3-str2 at https://github.com/z3str/Z3-str and the installation instructions at https://github.com/z3str/Z3-str/blob/master/README_OLD.md. 

- The TAC formulas of each path to a vulnerable sink have to be rewritten as Z3 specifications to verify the exploitability of the path. The solver package at https://github.com/aalhuz/navex/tree/master/src/solver encapsulates our translation to solver specification implementation. Specifically, run StaticSolver.java, and you will be prompted to enter the vulnerability type (i.e., SQL, XSS, etc.) you are investigating. This java program will do the following: 
  * Read both result files generated by the traversals.
  * Generate solver specification files for vulnerable paths. The spec files can be found in staticAnalysisSpec directory.
  * Invoke Z3-str2 to solve the constructed formulas in the Spec files. The models will also be in staticAnalysisSpec Directory. 
  * create a file that resolves the inclusion relationships to find candidate URLs (used in Step 2 as described in the paper). The file will be in results/include_map_resolution_results.txt. Note, paths to the above directories are hardcoded and have to be changed before running the program. 
 
 
# Step 2: concrete exploit generation #
Prerequisites: Deploy on a server (e.g., localhost) the applications that Step 1 found vulnerabilities in them (not all the applications that you have tested). Read more about this under "setup" in the evaluation section of our paper.
 Xdebug for trace generation is required too. We have used version 2.5.2 in our evaluation. Xdebug and its installation instructions are at https://xdebug.org/.

In this step, Navex crawls web applications to construct their navigation graphs (Neo4j graph). 


## The Navigation graph setup ##
- specify the name of the navigation graphthat we are about to construct in "org.neo4j.server.database.location" in $PATH_tO_YOUR_NEO4J_INSTALLATION/conf/neo4j-server.properties. 
For example: 
    
    org.neo4j.server.database.location = "$YOUR_PATH_TO_BATCH_IMPORT/navigationGraph.db"
    
Then, point to your Neo4J installation and start the server.
               			

## Application crawling ##
We have extended crawler4j in the fork https://github.com/aalhuz/crawler4j to allow for web forms and JavaScript reasoning.  
- To run extended crawler and construct the Navigation graph, run "run.pl" script in the Navex directory as the following 
            
            cd navex
           ./run.pl data 1 config/auth-appName.txt $SEED_URL 

config/auth-appName.txt is a file that you have to create to store login information for appName. A sample file is provided. $SEED_URL is the seed URL for the crawler (e.g., http://localhost/appName/index.php). While crawling the applications, nodes and edges will be added to the navigationGraph.db simultaneously. 

## Concrete exploit generation ##
To find navigation paths to exploit seeds. We have to traverse the Navigation graph using  exploitFinding.py in our python-joern fork. This wrapper script invokes traversals that check the inclusion map (in results/include_map_resolution_results.txt), matches it with the exploit seeds (i.e., exploit strings), and finally outputs the concrete exploits in results/navigation_sequences.txt. 
 
		    cd python-joern
		    python exploitFinding.py $ATTACK_TYPE
  For XSS, for instance, the $ATTACK_TYPE would be "xss" (python exploitFinding.py xss).

