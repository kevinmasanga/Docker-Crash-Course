##START AND STOP CONTAINERS
##Docker run command creats a new command and it dooesn't re use the previous containers
``docker ps -a 
     list all containers weather they are running or stopped
``
``docker start id
      starts a container that was stopped
``
``docker run --name 
     give the container a name
``
##Private registries
#A docker ###registry is a centralized service or platform that stores, manages and distributes container images acting as the hosting location 
#e.g Github but docker  ###repository is a collection within that registry containing related images with the ame name but differentversion tags

##Building Own Docker images
####Docker file--Build instructions

