# Part 1 - Selling CI/CD

## Objective

Explain the Fundamentals and Benefits of CI/CD to achieve build and deploy automation for Cloud-Based Software Products

Key Points:
- It should be good enough to submit to a manager in a real job.
- It should not be a messy or last-minute submission.
- You may use public domain or open source templates and graphics if you'd like.
- Your presentation should be no longer than 5 slides.

## Sample Structure

### Slide 1: Title and Premise

A title that catches the attention of the audience

**Example**: `CI/CD - A better way to build and ship products to market`

A subtitle that elaborates briefly on the title

**Example**: `Benefits of CI/CD to achieve build and deploy automation within our company's products`

### Slide 2: Continuous Integration
- Briefly describe CI in non-technical terms
- It's all about code and “Dev” workflow
- List some CI related phases: Build, Unit Test, Static Analysis

**Content Example**

```markdown
- The practice of merging all developers' working copies to a shared mainline several times a day. It's the process of "Making".

- Everything related to the code fits here, and it all culminates in the ultimate goal of CI: a high quality, deployable artifact!

- Some common CI-related phases might include:
    - Compile
    - Unit Test
    - Static Analysis
    - Dependency vulnerability testing
    - Store artifact
```

### Slide 3: Continuous Deployment
- Briefly describe CD in non-technical terms
- It's all about deployments and “Ops” workflow
- List some CD related phases: Provisioning Infrastructure, Configuring Infrastructure, Promotion, etc.

**Content Example**
```markdown
- A software engineering approach in which the value is delivered frequently through automated deployments.

- Everything related to deploying the artifact autonomously fits here. It's the process of "Moving" the artifact from the shelf to the spotlight without human intervention.

- Some common CD-related phases might include:
    - Creating and configuring infrastructure
    - Promoting to production
    - Smoke Testing (aka Verify)
    - Rollbacks in case if any failure
```

### Slides 4 and 5: Benefits of CI/CD at the Business Level

Try to cover the all the following benefits:
- Avoiding Cost
- Reducing Cost
- Increasing Revenue
- Protecting Revenue

You can refer to Lesson 2 Part 6 for benefits of CI/CD, format them as you see fit

**Content Example**
```markdown
- The rationale of CI/CD is the famous saying: ‘a penny saved is a penny earned', so here's a preview of the benefits of setting up a CI/CD pipeline:

    - Automate Infrastructure Creation:
        This will help to avoid cost by providing less human error, which means faster deployments

    - Faster and More Frequent Production Deployments:
        This would help to increase revenue by releasing new value-generating features more quickly

    - Automated Smoke Tests:
        This would help protect revenue by reducing downtime from a deploy-related crash or a major bug

    - Detect Security Vulnerabilities:
        This would help to avoid cost by preventing embarrassing or costly security holes.

    - Deploy to Production Without Manual Checks:
        This would help to increase revenue by making features take less time to market.

    - Detect Security Vulnerabilities:
        This would help to avoid cost by preventing embarrassing or costly security holes.

And many more …

```
