# CMPE 172 - Lab #10 Notes

- Tool dependencies are:
  - Gradle v5.6 
  - Java JDK 11

## Install Gradle using SDK Man:  https://sdkman.io/
- install sdk: `curl -s "https://get.sdkman.io" | bash`
![](images/6.png)
- start sdk: `source "$HOME/.sdkman/bin/sdkman-init.sh"`
- check if installation succeeded:`sdk version`
![](images/7.png)
- install gradle: `sdk install gradle`
![](images/8.png)
## CI Workflow (Part 1)
- https://help.github.com/actions/language-and-framework-guides/building-and-testing-java-with-gradle
### Using the Gradle starter workflow 
This part includes steps by steps and corresponding screenshots.
#### Creating workflow
1. Create a .github/workflows directory in your repository on GitHub if this directory does not already exist.
2. In the .github/workflows directory, create a file named gradle.yml. 
3. Copy the following YAML contents into the gradle.yml file:
- Set up workflow to trigger on push and pr on main branch and optionally upload build artifact 
````
name: Java CI with Gradle

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2
    - name: Set up JDK 11
      uses: actions/setup-java@v2
      with:
        java-version: '11'
        distribution: 'adopt'
    - name: Grant execute permission for gradlew
      run: chmod +x gradlew
    - name: Build with Gradle
      run: ./gradlew build
    - name: Build Result
      run: ls build/libs
    - name: Upload a Build Artifact
      uses: actions/upload-artifact@v2.2.3
      with:
        name: spring-gumball
        path: build/libs/spring-gumball-2.0.jar
````
![](images/1.png)
![](images/2.png)
4. Scroll to the bottom of the page and select Create a new branch for this commit and start a pull request. Then, to create a pull request, click Propose new file.
![](images/3.png)
5. Committing the workflow file to a branch in your repository triggers the push event and runs your workflow.
![](images/4.png)
![](images/5.png)
#### Viewing the workflow results
1. On GitHub.com, navigate to the main page of the repository. 
2. Under repository name, click  Actions.
3. In the left sidebar, click the workflow you want to see.
4. From the list of workflow runs, click the name of the run you want to see.
5. Under Jobs , click the Explore-GitHub-Actions job.
6. The log shows you how each of the steps was processed. Expand any of the steps to view its details.
- Error
![](images/9.png)
![](images/10.png)
- Success
![](images/14.png)
![](images/15.png)
## CD Workflow (Part 2)
- https://github.com/google-github-actions/setup-gcloud/tree/master/example-workflows/gke (Links to an external site.)
- https://cloud.google.com/iam/docs/creating-managing-service-accounts (Links to an external site.)
- https://kustomize.io (Links to an external site.)
  

- This is a workflow that uses GitHub Actions to deploy a spring-gumball app to an existing Google Kubernetes Engine cluster.
- For pushes to the main branch, this workflow will:
  - Download and configure the Google Cloud SDK with the provided credentials. 
  - Use a Kubernetes Deployment to push the image to the cluster. 
1. Check and modify the configuration for Google Kubernetes Engine cluster in repository
- kustomization.yml
````
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
````
- deployment.yaml
````
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-gumball-deployment
  namespace: default
spec:
  selector:
    matchLabels:
      name: spring-gumball
  replicas: 4 # tells deployment to run 2 pods matching the template
  template: # create pods using pod definition in this template
    metadata:
      # unlike pod.yaml, the name is not included in the meta data as a unique name is
      # generated from the deployment name
      labels:
        name: spring-gumball
    spec:
      containers:
      - name: spring-gumball
        image: gcr.io/PROJECT_ID/IMAGE:TAG
        ports:
        - containerPort: 8080
````
- service.yaml
````
apiVersion: v1
kind: Service
metadata:
  name: spring-gumball-service 
  namespace: default
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 8080 
  selector:
    name: spring-gumball
````
2. Set up secrets in your workspace:
   1. Create a new Google Cloud Project (or select an existing project) and enable the Container Registry and Kubernetes Engine APIs. 
       - Here I choose existing project "cmpe172"
   ![](images/11.png)
       - Enable the Container Registry and Kubernetes Engine APIs
   ![](images/12.png)
   2. Create a new GKE cluster or select an existing GKE cluster. 
       - Here I select existing GKE cluster
   ![](images/13.png)
   ![](images/16.png)
   3. Create or reuse a GitHub repository for the example workflow:
      - I use my GitHub repository: https://github.com/Diem-Vu/spring-gumball
   4. Create a Google Cloud service account
   - Enable the IAM API.
   ![](images/12.png)
   - On the gc menu slide on the left, choose "IAM & Admin"
   - Choose "Service Accounts"
   ![](images/17.png)
   ![](images/18.png)
   - Click "CREATE SERVICE ACCOUNT" to create a service account
   ![](images/19.png)
   ![](images/20.png)
   ![](images/22.png)
   5. Add the following Cloud IAM roles to service account:
      1. Kubernetes Engine Developer - allows deploying to GKE
   6. Create a JSON service account key for the service account.
   - Create key with JSON
     ![](images/23.png)
     ![](images/24.png)
     ![](images/25.png)
   7. Add the following secrets to repository's secrets:
   - GKE_PROJECT: Google Cloud project ID 
   - GKE_SA_KEY: the content of the service account JSON file 
   ![](images/26.png)
   ![](images/27.png)
   ![](images/28.png)
   ![](images/29.png)
3. Update .github/workflows/google.yml to match the values corresponding to VM:
- GKE_CLUSTER - the instance name of your cluster 
- GKE_ZONE - the zone your cluster resides
- Change the values for the GKE_ZONE, GKE_CLUSTER, IMAGE, and DEPLOYMENT_NAME environment variables (see example below).
````
name: Build and Deploy to GKE

on:
  release:
    types: [created]

env:
  PROJECT_ID: ${{ secrets.GKE_PROJECT }}
  GKE_CLUSTER: cmpe172    # TODO: update to cluster name
  GKE_ZONE: us-central1-c   # TODO: update to cluster zone
  DEPLOYMENT_NAME: spring-gumball # TODO: update to deployment name
  IMAGE: spring-gumball

jobs:
  setup-build-publish-deploy:
    name: Setup, Build, Publish, and Deploy
    runs-on: ubuntu-latest
    environment: production

    steps:
    - name: Checkout
      uses: actions/checkout@v2

    # Build JAR File
    - name: Set up JDK 11
      uses: actions/setup-java@v2
      with:
        java-version: '11'
        distribution: 'adopt'
    - name: Grant execute permission for gradlew
      run: chmod +x gradlew
    - name: Build with Gradle
      run: ./gradlew build
    - name: Build Result
      run: ls build/libs

    # Setup gcloud CLI
    - uses: google-github-actions/setup-gcloud@v0.2.0
      with:
        service_account_key: ${{ secrets.GKE_SA_KEY }}
        project_id: ${{ secrets.GKE_PROJECT }}

    # Configure Docker to use the gcloud command-line tool as a credential
    # helper for authentication
    - run: |-
        gcloud --quiet auth configure-docker

    # Get the GKE credentials so we can deploy to the cluster
    - uses: google-github-actions/get-gke-credentials@v0.2.1
      with:
        cluster_name: ${{ env.GKE_CLUSTER }}
        location: ${{ env.GKE_ZONE }}
        credentials: ${{ secrets.GKE_SA_KEY }}

    # Build the Docker image
    - name: Build
      run: |-
        docker build \
          --tag "gcr.io/$PROJECT_ID/$IMAGE:$GITHUB_SHA" \
          --build-arg GITHUB_SHA="$GITHUB_SHA" \
          --build-arg GITHUB_REF="$GITHUB_REF" \
          .

    # Push the Docker image to Google Container Registry
    - name: Publish
      run: |-
        docker push "gcr.io/$PROJECT_ID/$IMAGE:$GITHUB_SHA"

    # Set up kustomize
    - name: Set up Kustomize
      run: |-
        curl -sfLo kustomize https://github.com/kubernetes-sigs/kustomize/releases/download/v3.1.0/kustomize_3.1.0_linux_amd64
        chmod u+x ./kustomize

    # Deploy the Docker image to the GKE cluster
    - name: Deploy
      run: |-
        ./kustomize edit set image gcr.io/PROJECT_ID/IMAGE:TAG=gcr.io/$PROJECT_ID/$IMAGE:$GITHUB_SHA
        ./kustomize build . | kubectl apply -f -
        #kubectl rollout status deployment/$DEPLOYMENT_NAME
        #kubectl get services -o wide
````
![](images/30.png)
4. Trigger a CD Deployment by creating a new GitHub Release
- Click "Create a new release"
![](images/31.png)
- Fill release tag and title
![](images/32.png)
- Publish release
![](images/33.png)
- Build and Deploy to GKE 
![](images/39.png)
![](images/34.png)
![](images/35.png)
![](images/36.png)
![](images/40.png)
![](images/41.png)
- Create Ingress
![](images/42.png)
![](images/43.png)
- Web UI should come up on Load Balancer's External IP
![](images/44.png)

## Resources (online blogs or troubleshooting tips) 
- Understanding GitHub Actions: https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions
- Solve for chmod +x gradlew cannot access 'gradlew' when run workflow gradle.yml:
  - https://github.com/google-github-actions/setup-gcloud/tree/main/example-workflows/gke/.github/workflows
  - https://github.com/google-github-actions/setup-gcloud/blob/main/example-workflows/gke/.github/workflows/gke.yml
  - https://github.com/gradle/gradle/commit/d96d1f21cfa94683fe2557bd468d39fcf2ebe038
  - https://github.com/gradle/gradle
  - https://stackoverflow.com/questions/17668265/gradlew-permission-denied
  - https://www.codegrepper.com/code-examples/whatever/run+chmod+%2Bx+gradlew+chmod%3A+cannot+access+%27gradlew%27%3A+no+such+file+or+directory+error%3A+process+completed+with+exit+code+1.
  - https://discuss.gradle.org/t/adding-gradle-wrapper-files-to-gitignore/27428
  - https://github.community/t/gradlew-no-such-file-or-directory/17976
- Locate the project ID: https://support.google.com/googleapi/answer/7014113?hl=en
## Errors & Resolve Methods
- CI Workflow (Part 1)
  - Error: cannot access 'gradlew' when run workflow gradle.yml
![](images/gradle.png)
  - Solve: I search for the topic in google.com, ask people by posting the error description to class discord and slack. Then I tried all methods that I can think about: 
    - I checked if the directory is correct (helpful)
    - I check if needed files are in repo (helpful)
      - gradlew
      - gradlew.bat
      - gradle/wrapper/gradle-wrapper.jar
      - gradle/wrapper/gradle-wrapper.properties
    - I re-run jobs (helpful)
    - I re-created thw workflow in different ways: (not helpful)
      - create new workflow for "Java and Gradle" using default yml file, modified yml file
      - create yml file then pull request as new branch
      - create yml file then pull request and merge the branch
      - create yml file directly to main
    - I modified gradle.yml (Not helpful)
      - `./gradlew build` to 'gradle' build
      - create wrapper by adding 
    ````
    - name: create wrapper
      run: gradle wrapper
    ````
    - I clone professor's git repo, copied it, paste to my repo, push my repo again. Then I re-run the workflow (helpful)
    - I checked the tree: (helpful)
    ````
    .
    ├── build.gradle
    ├── settings.gradle
    ├── gradle
    │ └── wrapper
    │ ├── gradle-wrapper.jar
    │ └── gradle-wrapper.properties
    ├── gradlew
    └── gradlew.bat
    ````

    - I clicked to edit the file gradle.yml and commit it directed to main (helpful)
- CD Workflow (Part 2)
  - Error: Setting project Id from credentials: Required "container.clusters.get" permission(s) for "projects/***/zones/us-central1-c/clusters/cmpe172".
![](images/gke1.png)
![](images/gke2.png)
  - Solve: There was someone in the class had the same issues, and I got solution from them
    - Grant owner role to the service (Fail)
      - ![](images/gke3.png)
      - ![](images/gke4.png)
    - Rebuild a new service and granting owner role when creating (Success)
      - ![](images/gke5.png)