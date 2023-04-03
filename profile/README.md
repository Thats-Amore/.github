[![Thats Amore](https://raw.githubusercontent.com/Thats-Amore/.github/main/profile/header.svg)](#)

> That's Amore is a new tech startup venture out of Austin, TX
> hoping to connect hungry consumers with real-time insights about the fastest ways they can order and receive their favorite types of pizza from local restaurants.
>
> Armed with an aggressive marketing plan,
> the service is expected to serve hundreds of thousands of customers as soon as the platform's product moves to an initial release stage.
> Development is fiercely ongoing.
>
> During this time of preparation, the company is seeking to hire an infrastructure engineer who can manage and deploy a fully-cloud,
> scalable infrastructure, to meet demand at peak times while also saving costs during off-hours when demand is low.
> As a potential candidate of this role, you've been given a mission:
> considering That's Amore's startup position and its expectations for the future,
> how would you best design the company's cloud infrastructure?

The company believes they need an infrastructure engineer, but the first thing I'd advise is that they don't want an *infrastructure engineer*,
they want a *DevOps engineer*.
The title itself doesn't matter, but today you have to think like a *developer who happens to be working on infrastructure*.
Gone are the days of a hundred undocumented manual steps to spin something up, deployments need to be automated and reproducible.


# First thoughts about prompts:

With that being said, to start here are some of my thoughts to the prompts:


  1. > Developers currently deploy all changes to a single hosted "Production" environment (with just a few servers) after testing and auditing changes locally.
     > Would you change this?
     > If so, what would you do?

     Instead of deploying all changes to a single hosted production environment,
     I'd split environments into separate AWS accounts, at a minimum something like:
       * dev
       * stage
       * prod

     Resources would be defined by Terraform modules, which would be deployed into one of the above environments
     by using Terragrunt (a thin wrapper for Terraform).
     Terragrunt will define the variables specific to an AWS account/environment and let us use a specific version of a Terraform module from git
     so we know exactly what is deployed.

     At first resources would be "manually" deployed with Terragrunt where necessary but
     setting up a continuous integration/continuous deployment (CI/CD) pipeline for deploying services is the longer term goal.


  1. > Developers are frustrated by their deployment toolchain/process,
     > so they often circumvent policies to get their changes deployed.
     > What would you do about this?

     The first thing to do here is determine what frustrations there are with the deployment toolchain/process.
     I would work with the development team to understand their pain points and identify ways to streamline the process.

     Given the fast paced nature of a startup, I would not immediately try to enforce strict controls on the process,
     there are (rare) circumstances where it is necessary to bypass the normal process.
     The more important thing is to ensure that one team cannot touch the resources of another team, and that
     you have backup and disaster recovery processes in place.

     Once the processes have matured and you have a working CI/CD pipeline you can consider locking them down,
     but even then there should be a way for a privileged user to deploy something directly to prod in an emergency situation.


  1. > When it’s time to scale up, how can the company be certain that the infrastructure is the same across any given host in a particular environment at one time?
     > For example, if a user sends multiple web requests to different webservers for API requests,
     > the responses must be consistent and reliable.

     This question has two parts:
       * Ensuring the software running is the same across environments.
       * Ensuring that the actual data (i.e. whats in the database) is in sync.

     To address the first part, we ensure that the software running is the same by only deploying via
     automated processes, i.e. we deploy with Terragrunt and use a tagged release of a Terraform module,
     or for Kubernetes we deploy a tagged release of a helm chart, etc.

     For data consistency, a lot of the time we don't actually care if there is a little lag behind showing consistent results.
     Where we can't afford to have stale data then we must explicitly request a strongly consistent read from the database.



  1. > There's a lot going on at any given time.
     > Which tools would you equip yourself with to ensure your tasks are easily manageable?

     For my personal use, I just use plaintext notes that I keep track of with git.
     However, for collaboration with a team I may also rely on something like Jira, Trello, a spreadsheet, etc.

     As for documentation, it should live as close to the code as possible, e.g. a markdown file in the same git repo, or inline comments.
     And documentation should not be so verbose that small details become forgotten and go without updates, it should be as brief as possible
     to convey a point but not so exhaustive that updating it takes longer than an update to the code took itself.


  1. > All company employees access critical resources (marketing materials, corporate branding images, etc.)
     > through a single cloud endpoint with little to no access controls.
     > Should you change this? If so, how?

     To start I'd move this endpoint to an internal IP that can only be accessed while connected to a VPN.
     Then, to improve access control and security,
     I would implement an identity and access management (IAM) system to enforce granular permissions for accessing critical resources.
     This would allow the company to control who has access to sensitive information and prevent unauthorized access.


  1. > What's your strategy for disaster prevention and mitigation?

     I'd implement a disaster recovery plan that includes regular backups of critical data, likely as tar files into S3.
     One good way to make sure your disaster recovery process actually works is to use the process to move prod data
     into a lower staging environment for testing.
     If you perform that process regularly to have prod-like data in stage, then a nice side affect is that you become very familiar
     with the recovery process so in a worst case scenario you'd know how to restore that prod data back into prod.

     However, there are many mechanisms available to enable us to easily recover from simple failures automatically.
     I'd architect services to be deployed across multiple availability zones and implement failover mechanisms
     like automatic scaling policies to ensure that the system can handle sudden spikes in traffic or failures of individual components.
     Automatic scaling isn't just to handle changes in load, but also if something goes wrong a node can be killed off and a new one brought up in it's place.
     Even for a simple non-ha service on a single EC2, you can put it in an auto scaling group so that if a health check fails the old EC2 is killed and a new
     one is spun up without needing any manual intervention.

# Plan & Code Snippets

Given the need to move quickly in a startup, I'd likely not waste the time to implement all of the initial infrastructure boilerplate over several months.
There are great reference architectures available, I'd use one of these to get going so that time can be better spent on developing the infrastructure
specific to the organization.

It could be tempting to just deploy something quickly without all of the other supporting resources in place
but it's worth configuring the networking/VPCs and access controls first
so that if you make a mistake later you're less likely to expose something publicly that you didn't mean to.

This diagram from Gruntworks shows a good example of a multi account setup. I would implement something similar.

[![ref](https://raw.githubusercontent.com/Thats-Amore/.github/main/profile/gruntwork-landing-zone-ref-arch.png)](#)


Since the prompt mentioned that the team already has a product they deploy to a handful of servers,
I'd work to adapt it to an EC2 based service with an auto scaling group so it could be quickly deployed into the new cloud environment.

As an example I've setup two git repos within this github organization:
  * [infra_envs](https://github.com/Thats-Amore/infra_envs) - This repo contains the infrastructure for each environment/account
  * [terraform_modules](https://github.com/Thats-Amore/terraform_modules) - This repo contains re-usable terraform modules
