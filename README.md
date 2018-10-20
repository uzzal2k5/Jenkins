# Jenkins

This pipeline, I create for one of my client. For Security reason, I do not keept the original credentialsID into Jenkinsfile. I use  and arbitrary name as a credentials ID.

### What Is Done Here !
Cloning two seprate repository, Cloning trigger on commit to bitbucket. Build with maven using sonar build to analysis with sonaQube. After build the project, push artifacts to Nexux Artifactory
And finally deploy to Cloud Foundry
Buidl the projects under certain directory.

A notification through Email has been create to notify starting Jenkis Job as well as when finish with success. This notification notify if the build fail with log links


