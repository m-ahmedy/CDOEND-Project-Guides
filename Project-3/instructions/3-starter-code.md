# Part 3 - Starter Code

- The [starter code](https://github.com/udacity/cdond-c3-projectstarter) of the project is provided in the project instructions on your classroom

- Clone the starter code locally

```bash
git clone https://github.com/udacity/cdond-c3-projectstarter
```

## Project Structure

```markdown
├── .circleci 
│   ├── ansible
│   └── files
├── backend
│   ├── src
│   └── test
├── frontend
│   ├── src
│   └── types
└── util
```

## Configuration folder: ".circleci"

- Contents of the ".circleci" directory
    - **files**: contains CloudFormation templates:
        - **cloudfront.yml** - We will instruct you to use this file to manually create an initial stack.
        - **backend.yml** and **frontend.yml** - Your CircleCI job will run these files. We will hold you along the step-by-step instructions ahead.

        - Note that **no changes** are required in three files mentioned above, **except** updating the EC2 **key pair name** and **suitable AMI ID** in the **backend.yml** file later.
    - **ansible**: contains Ansible Playbooks and Roles, most of them are incomplete, so they will need to be updated in appropriate stages
        
        Contents of the "ansible" folder
        - Playbooks: at "**.circleci/ansible**"
            - Configure Server (Incomplete)
            - Deploy Backend (Incomplete)
        - Roles: at "**.circleci/ansible/roles**"
            - Configure Server (Incomplete)
            - Deploy (Incomplete)
            - Configure Prometheus Node Exporter (Complete)

    - **CircleCI config.yml** file:
        This file has intentionally failing/incomplete jobs.
        To call attention to **unfinished** jobs, there are some "**non-zero error codes**" (e.g. **exit 1**).
        The starter config.yml file will not work out-of-the-box.

### Source code folders: backend/ and frontend
It is a good practice to read through the frontend and backend README.

Note that there are intentional errors left in the source code, and it **will not work out of the box on your local machine**

The intentional errors are	
- One compile error in the backend. 
- One failing test in both the frontend and backend.

#### Backend README.md

Notes to take from backend/README.md:
- Scripts:
    - Build: `npm run build`
    - Unit Test: `npm run test`
    - Scan for vulnerabilities: `npm run audit --audit-level=critical`
    - Run DB migrations: `npm run migrations`
    - Revert DB migrations: `npm run migrations:revert`

- Environment variables:
You need to add some environment variables to your system in order to run migrations or the backend.

- **TYPEORM_CONNECTION**: The type of the database, always set to `postgres`
- **TYPEORM_HOST**: The endpoint of the DB instance, set to either values:
    - The RDS instance **endpoint** for production
    - or `localhost` for developing locally with Docker Compose
- **TYPEORM_PORT**: The port where DB communication will occur, set to either values:
    - `5423`: for production RDS instance
    - `5532`: for local development using the provided Docker Compose file
- **TYPEORM_USERNAME**: Username of the DB
    - **The username you set in RDS**: for production RDS instance
    - `postgres`: for development using the provided Docker Compose file
- **TYPEORM_PASSWORD**: Password of the DB
    - `password` for local Docker Compose development
    - **The password you set in RDS** for production
- **TYPEORM_DATABASE**: The actual database running on the instance
    - This is **not the RDS Instance Identifier**, but it's the **Initial database name**, make sure not to leave this field empty when creating the RDS instance
    - Keep it `glee` for both development and production
- **TYPEORM_MIGRATIONS**: Where DB migrations are located in the code
    - Set to `./src/migrations/*.ts`
    - We will change this value when deploying the production build of the backend on EC2
- **TYPEORM_MIGRATIONS_DIR**: Where DB migrations directory is located in the code
    - Set to `./src/migrations`
    - We will change this value when deploying the production build of the backend on EC2
- **TYPEORM_ENTITIES**: Where DB entities are located in the code
    - Set to `./src/modules/**/*.entity.ts`
    - We will change this value when deploying the production build of the backend on EC2

#### Backend README.md

Notes to take from backend/README.md:
- Scripts:
    - Build: `npm run build`
    - Unit Test: `npm run test`
    - Scan for vulnerabilities: `npm run audit --audit-level=critical`

- Environment variables:
You need to add one environment variable to your system in order to run migrations or the backend.

- **API_URL**: The backend URL which is hosting the application
    - `http://localhost:3030`: For local development
    - `http://<EC2 Public IP>:3030`: For production backend build

## Instructions
1. Create a new repo on GitHub, name it whatever you see fit

2. Initialize a local Git repo on a folder in your local machine
    ```
    # In a folder you created for the project
    git init
    ```

3. Connect your local git repo with GitHub
    ```
    git remote add <GitHub Repo URL>
    ```

4. Create a basic `README.md` file

5. Copy starter files from the starter code repo, you'll need only, `.circleci`, `backend`, `frontend`, `.gitignore`, and `util` if you're planning for local development

6. Commit and push the changes to the `master` branch
    ```
    git add .
    git commit -m "Initial commit"
    git push -u origin master
    ```

7. Open CircleCI and create a project for the GitHub repo
