# Part 4 - CI Stages

## Objectives

**Complete CI Stages**

- Build
  - Fix the compilation error in the backend source code
- Unit Test
  - Fix the failing test case in the backend source code
  - Fix the failing test case in the frontend source code
- Scan for Critical Vulnerabilities
  - Fixing critically vulnerable packages in the backend source code
  - Working around other critical vulnerabilities

**Submission requirements**

- Job failed because of compile errors. [**SCREENSHOT01**]
- Job failed because of unit tests. [**SCREENSHOT02**]
- Job that failed because of vulnerable packages. [**SCREENSHOT03**]

**Affected Files**

- CircleCI config: `.circleci/config.yml`
- Backend Source Code
  - `backend/src/main.ts`
  - `backend/src/modules/domain/employees/commands/handlers/employee-activator.handler.spec.ts`
  - `backend/package.json`
- Frontend Source Code
  - `frontend/src/app/components/LoadingMessage/LoadingMessage.spec.tsx`

## Implementation

### General Structure

All CI stages need a base docker image that supports **Node.js 13.8.0** and CircleCI

We will use [`cimg/node:13.8.0`](https://circleci.com/developer/images/image/cimg/node) docker image, it's a convenience image provided by CircleCI that's built to run Node.js 13.8.0 on CircleCI

So the executor environment will be the same for all upcoming stages in this guide

```yml
jobs:
  ...
  JOB:
    docker:
    - image: cimg/node:13.8.0
```

In addition to the executor environment, all CI stages has common structure of steps to be executed

1. Checkout the code from the repository

   ```yml
   - checkout
   ```

1. `restore_cache`: The built-in feature in CircleCI to cache dependencies so that it doesn't take much time

   ```yml
   - restore_cache:
       keys:
         - TIER-deps-{{ checksum "TIER/package-lock.json" }}
   ```

   Where `TIER` can only be either **frontend** or **backend**

   _Note_: The cache key utilizes the current state of the dependencies that's stored in the lock file `package-lock.json`, any changes to this file will result in creating a new cache key

1. Install dependencies from NPM

   ```yml
   - run:
       name: Install dependencies
       command: |
         cd TIER
         npm install
   ```

   Where `TIER` is either **frontend** or **backend**

1. Run the needed script

   - Build stage

     ```yml
     - run:
         name: Build TIER
         command: |
           cd TIER
           npm run build
     ```

   - Test stage

     ```yml
     - run:
         name: Run TIER unit tests
         command: |
           cd TIER
           npm run test
     ```

   - Scan stage

     ```yml
     - run:
         name: Scan TIER packages
         command: |
           cd TIER
           npm audit --audit-level=critical
     ```

1. **Only for Build stages**: Save the dependencies to cache, if it does not exist

   ```yml
   - save_cache:
       paths: [TIER/node_modules]
       key: TIER-deps-{{ checksum "TIER/package-lock.json" }}
   ```

### Build stage

#### Jobs: build-frontend

Following the instructions at the beginning of this guide will yield the following configuration for the `build-frontend` stage

```yml
jobs:
  build-frontend:
    docker:
      - image: cimg/node:13.8.0
    steps:
      - checkout
      - restore_cache:
          keys:
            - frontend-deps-{{ checksum "frontend/package-lock.json" }}
      - run:
          name: Install dependencies
          command: |
            cd frontend
            npm install
      - run:
          name: Build frontend
          command: |
            cd frontend
            npm run build
      - save_cache:
          paths: [frontend/node_modules]
          key: frontend-deps-{{ checksum "frontend/package-lock.json" }}
```

#### Jobs: build-backend

Similarly, following the instructions at the beginning of this guide will yield the following configuration for the `build-frontend` stage

```yml
jobs:
  ...
  build-backend:
    docker:
    - image: cimg/node:13.8.0
    steps:
    - checkout
    - restore_cache:
       keys:
         - backend-deps-{{ checksum "backend/package-lock.json" }}
    - run:
        name: Install dependencies
        command: |
          cd backend
          npm install
    - run:
        name: Build backend
        command: |
          cd backend
          npm run build
    - save_cache:
        paths: [backend/node_modules]
        key: backend-deps-{{ checksum "backend/package-lock.json" }}
```

#### Workflow update

Then update the workflow at the end of config.yml

```yml
workflows:
  default:
    jobs:
      - build-frontend
      - build-backend
```

---

Commit and push the changes to GitHub to trigger a workflow on CircleCI

---

A new workflow will be triggered on CircleCI

The `build-backend` job will fail due to the **intentional** compile error

Take a screenshot of the error on CircleCI, it should be something like this, this will be [**SCREENSHOT01**]

![SCREENSHOT01](../assets/Screenshot-1.png)

#### Fixing the error

##### Backend Compile Error Fix

Go to the specified compile error location - `backend/src/main.ts` line `31` - and remove the letter `x` and the comment

**Before**

```js
    .addBearerAuth()x // here is an intentional compile error. Remove the "x" and the backend should compile.
```

**After**

```js
    .addBearerAuth()
```

---

Commit the change

---

### Test stage

#### Jobs: test-frontend

Following the instructions at the beginning of this guide will yield the following configuration for the `test-frontend` stage

```yml
jobs:
  ...
  test-frontend:
    docker:
    - image: cimg/node:13.8.0
    steps:
    - checkout
    - restore_cache:
       keys:
         - frontend-deps-{{ checksum "frontend/package-lock.json" }}
    - run:
        name: Install dependencies
        command: |
          cd frontend
          npm install
    - run:
        name: Run frontend unit tests
        command: |
          cd frontend
          npm run test
```

#### Jobs: test-backend

Similarly, following the instructions at the beginning of this guide will yield the following configuration for the `test-frontend` stage

```yml
jobs:
  ...
  test-backend:
    docker:
    - image: cimg/node:13.8.0
    steps:
    - checkout
    - restore_cache:
       keys:
         - backend-deps-{{ checksum "backend/package-lock.json" }}
    - run:
        name: Install dependencies
        command: |
          cd backend
          npm install
    - run:
        name: Run backend unit tests
        command: |
          cd backend
          npm run test
```

#### Workflow update

Then update the workflow at the end of config.yml

```yml
workflows:
  default:
    jobs:
      - build-frontend
      - build-backend
      - test-frontend:
          requires: [build-frontend]
      - test-backend:
          requires: [build-backend]
```

---

Commit and push the changes to GitHub to trigger a workflow on CircleCI

---

A new workflow will be triggered on CircleCI

Both `test-frontend` and `test-backend` jobs will fail due to the **intentionally** failing unit tests

Take a screenshot of one of the errors on CircleCI, it should be something like these, this will be [**SCREENSHOT02**]

![SCREENSHOT02 Frontend](../assets/Screenshot-2-frontend.png)

![SCREENSHOT02 Backend](../assets/Screenshot-2-backend.png)

#### Fixing the errors

##### Frontend Unit Test fix

Go to the failing unit test suite location - `frontend/src/app/components/LoadingMessage/LoadingMessage.spec.tsx` line `11` - and remove the question mark `?` and the comment

**Before**

```js
expect(wrapper.contains(<span>{message}?</span>)).toBeTruthy(); //remove the question mark to make the test pass
```

**After**

```js
expect(wrapper.contains(<span>{message}</span>)).toBeTruthy();
```

---

Commit the change

---

##### Backend Unit Test fix

Go to the failing unit test suite location - `backend/src/modules/domain/employees/commands/handlers/employee-activator.handler.spec.ts` line `22` - and change `101` to `100`, and remove the comment

**Before**

```js
const params = {
  employeeId: 101, //change this to 100 to make the test pass
  isActive: false,
};
```

**After**

```js
const params = {
  employeeId: 100,
  isActive: false,
};
```

---

Commit the change

---

### Analyze stage

#### Jobs: scan-frontend

Following the instructions at the beginning of this guide will yield the following configuration for the `scan-frontend` stage

```yml
jobs:
  ...
  scan-frontend:
    docker:
    - image: cimg/node:13.8.0
    steps:
    - checkout
    - restore_cache:
       keys:
         - frontend-deps-{{ checksum "frontend/package-lock.json" }}
    - run:
        name: Install dependencies
        command: |
          cd frontend
          npm install
    - run:
        name: Scan frontend packages
        command: |
          cd frontend
          npm audit --audit-level=critical
```

#### Jobs: scan-backend

Similarly, following the instructions at the beginning of this guide will yield the following configuration for the `scan-frontend` stage

```yml
jobs:
  ...
  scan-backend:
    docker:
    - image: cimg/node:13.8.0
    steps:
    - checkout
    - restore_cache:
       keys:
         - backend-deps-{{ checksum "backend/package-lock.json" }}
    - run:
        name: Install dependencies
        command: |
          cd backend
          npm install
    - run:
        name: Scan backend packages
        command: |
          cd backend
          npm audit --audit-level=critical
```

#### Workflow update

Then update the workflow at the end of config.yml

```yml
workflows:
  default:
    jobs:
      - build-frontend
      - build-backend
      - test-frontend:
          requires: [build-frontend]
      - test-backend:
          requires: [build-backend]
      - scan-frontend:
          requires: [build-frontend]
      - scan-backend:
          requires: [build-backend]
```

---

Commit and push the changes to GitHub to trigger a workflow on CircleCI

---

A new workflow will be triggered on CircleCI

Both `scan-frontend` and `scan-backend` jobs will fail due to the vulnerable packages

Take a screenshot of one of the errors on CircleCI, it should be something like these, this will be [**SCREENSHOT03**]

![SCREENSHOT03 Frontend](../assets/Screenshot-3-frontend.png)

![SCREENSHOT03 Backend](../assets/Screenshot-3-backend.png)

#### Fixing the errors

The process of fixing vulnerabilities found in the project dependencies **is not** the responsibility of the DevOps engineer, and it **is** out of scope of the CI/CD pipeline

However, we will need our pipeline to pass, so we will use the `force fix` workaround

The `force fix` workaround **does not** fix the vulnerabilities permanently, but **only during the CI/CD pipeline** and **used only in the scan-\* stages**

The command we will use for the `force fix` workaround is

```bash
npm audit fix --force --audit-level=critical
```

##### Frontend Vulnerabilities Workaround Fix

Update the job in `.circleci/config.yml` to add the `force fix` command **just before** the audit command

_Note_: Add as many of this command to force-fix any remaining critical vulnerabilities

**Before**

```yaml
- run:
    name: Scan frontend packages
    command: |
      cd frontend
      npm audit --audit-level=critical
```

**After**

```yaml
- run:
    name: Scan frontend packages
    command: |
      cd frontend
      npm audit fix --force --audit-level=critical
      npm audit --audit-level=critical
```

---

Commit the change

---

##### Backend Vulnerabilities Workaround Fix

First, update the job in `.circleci/config.yml` to add the `force fix` command **just before** the audit command

Backend requires the workaround **twice**

_Note_: Add as many of this command to force-fix any remaining critical vulnerabilities

**Before**

```yaml
- run:
    name: Analyze backend
    command: |
      cd backend
      npm install
      npm audit --audit-level=critical
```

**After**

```yaml
- run:
    name: Analyze backend
    command: |
      cd backend
      npm install
      npm audit fix --force --audit-level=critical
      npm audit fix --force --audit-level=critical
      npm audit --audit-level=critical
```

Second, update `backend/package.json` with the following values

**Before**

```json
"class-validator": "^0.9.1",
```

**After**

```json
"class-validator": "0.12.2",
```

Also

**Before**

```json
"standard-version": "^4.4.0",
```

**After**

```json
"standard-version": "^7.0.0",
```

---

Commit the change

---

## Footnotes

After finalizing these stages the source code is ready for us to examine and see locally
