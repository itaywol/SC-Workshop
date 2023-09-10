# Securing Your Software Supply Chain Workshop

Welcome to the Software Supply Chain Workshop, focused on enhancing your skills in securing the software supply chain using open source tools. In this hands-on workshop, participants will engage in practical exercises designed to simulate real-world scenarios involving breached systems, vulnerabilities, and their corresponding solutions. Through these exercises, you'll learn how to identify, mitigate, and defend against security threats within the software supply chain.

## Prerequisite
 - Fork the repository into your Organization
 - On your Forked Repository Go to "Settings" Tab
 - Go to Pages and in the "Source" field choose "Github actions"
 - Go to "Actions" Tab and click "I understand my workflows, go ahead and enable them"
 - Go to "Deploy Page", click on "Run Workflow" and choose from "main" branch
 - Go to your Organization settings and choose Action->General
 - Under Workflow choose `Allow GitHub Actions to create and approve pull requests`
 - Wait 2 min for the Action Run to be completed
 - Go to the link under the deploy step, or directly to this address `https://<USER>.github.io/<REPOSITORY>`
 - Make sure "you have been pwned"


## Phase 1: Secrets Exposed

### Scenario
In this phase, we'll explore a scenario where a secret of a stale collaborator has been exposed within the repository, similar to the Toyota data breach incident mentioned in [this article](https://www.spiceworks.com/it-security/data-security/news/toyota-data-breach). Your task is to identify and address this security breach.

### Tasks
1. **Find and Revoke Secrets**
   - Create a new pipeline named `01.01 - Exposed Credential.yml` that scan the repository to locate exposed secrets with Trivy Action
```
name: 01.01 - Detect Exposed Credential
on: 
  workflow_dispatch

jobs:
  build:
    name: Secret Scanner
    runs-on: ubuntu-latest
    env:
      IMAGE_NAME: ${{ github.repository }}
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Run secret scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: "fs"
        format: 'table'
        exit-code: '0'
        scanners: 'secret'
```
   - Run the new workflow and find the credential within the workflow logs

2. **Implement Secret Scanner**
   - Configure this workflow to run on every PR Creation based `main` branch by changing the `on` field to:
```
on: 
  pull_request: 
         branches: ["main"]
  workflow_dispatch:
```
   - Configure the scanner to fail the build if any secrets are detected by changine the `exit-code` to `1`

3. **Strengthen Main Branch Protection**
   - Configure branch protection to enforce and block secrets on every merge request to `main` branch
     
4. **Remove Compromised Secret**
   - Make necessary changes to remove the compromised secret from the `main` branch.

## Phase 2: Compromised CI/CD

### Scenario
In this phase, we'll delve into a scenario where an attacker leverage the exposed credential to manipulates an existing build pipeline to introduce malicious changes to the release. Your objective is to detect and remediate this compromise.

### Tasks
1. **Implement Tracee in CI/CD**
   - Integrate Tracee into the [deploy-gh-pages.yml](.github/workflows/deploy-gh-pages.yml) workflow to detect malicious activities.
     - create a new branch
     - add start tracee step after the `Checkout` step name
   ```
      - name: Start Tracee profiling in background
        uses: aquasecurity/tracee-action@b5ad0734366c98eca9f76bd7a9d27368f837e1e7
   ```
     - add stop after the `Upload artifact` step name, and allow pr creation when there is profile updates
   ```
      - name: Stop and Check Tracee results and create a PR
        uses: aquasecurity/tracee-action@ce4a8e48ea62953976e7e10fcd83344bb85d132a
        with:
          create-pr: "true"
          fail-on-diff: "false"
   ```
     - Add permission to PR creation by adding `pull-requests: write`  under permission:
   ```
   permissions:
     contents: write
     pages: write
     id-token: write
     pull-requests: write
   ```
    - push chances to branch & create the PR
   

2. **Identify and Fix Malicious Payload**
   - Analyze the PR commnets to identify the introduced malicious payload.
   - Analyze the profiles in the new `Updates to tracee profile` PR
   - Fix the malicious parts
   - Analyze profiles files again to verify the malicious payloads removed
   - Merge both PRs

3. **Analyze The Malicous Source**
   - Investigate the source of the miner payload, and find the malicous commit in the dependency repository
     
4. **Enforce Signed Commits**
   - Restrict the acceptance of commits to the main branch to only those signed with approved keys.

5. **Bonus - Poison Dependency**
   - Create a new Github Action with legit content
   - Publish it
   - Add it into your [deploy-gh-pages.yml](.github/workflows/deploy-gh-pages.yml) with the Action Tag
   - Review the PR comments and the profile updates & merge
   - In your brand new github action Add a new command that creating new files\override exsiting ones
   - Push it into the exsiting version
   - In your workshop repository create any new change and Open PR
   - Make sure you see the both PR comments with signiture and Profile deviation updates

## Phase 3: Critical Vulnerability Emerged
### Scenario
In this phase, we'll delve into a scenario where an emerging critical vulnrability published and we are in a rush to identify where it used and mitigate the risk.

### Tasks
1. **Address Dependency Vulnerability**
   - Create a new workflow that run Trivy Action for every PR Creation and fails if there are Medium to Critical Vulnerabilties
```
name: 03.01 - Run Vulnrabilities Scanning
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
jobs:
  build:
    name: Vulnerabilities-Scanner
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Run vulnerabilities scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: "fs"
        format: 'table'
        ignore-unfixed: true
        scanners: 'vuln'
        exit-code: '1'
        severity: 'CRITICAL,HIGH,MEDIUM'
```
   - Fix vulnerable dependencies in `package.json` and don't forget to update `package-lock.json` by `npm i --package-lock` inside `my-app` directory

2. **Vulnerability Checks for PRs**
   - Add to the branch protection enforcment for vulnerability scanner
3. **Upload results to Security Tab**
   - add proper permission:
```
permissions:
  security-events: write
```
   - Change trivy-action format to `sarif` and set the `output` file
   - After the scanning part add a new step to upload Trivy results to Security Tab
```
    - name: Upload Trivy scan results to GitHub Security tab
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
        category: Trivy
```
4. **Check Security Tab**
5. **Bonus - Add a cron job trigger that will run the vulnerability scanner every night - Bonus**
6. **Enfore Code Reviews**
   - Add to the branch protection rule on the main branch, request to review and approve by other contributors before changes are merged.
