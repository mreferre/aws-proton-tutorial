### What is Proton?
Proton is a fully managed AWS service that helps engineering platform teams to build developer portals to streamline the SDLC (software development lifecycle). Proton has two main goals: increase developers' productivity and agility while allowing organizations to maintain the right level of control and governance. 

### Who are the Proton users? 
Proton has two main actors:
- the platform admin: responsible for authoring application templates and everything in the DevOps spectrum that aims at removing undifferentiated heavy lifting for the developer
- the developer: responsible for writing application code and deploying it using the platform abstractions and artifacts made available by the platform admin
> These two actors are mapped to two AWS managed IAM policies: `AWSProtonFullAccess` and `AWSProtonDeveloperAccess` (there is also a `AWSProtonReadOnlyAccess`).

### Why would you use Proton?  
There are two set of reasons why customers find Proton useful. They are grouped by users profiles. Usually the larger the organization is the most value they can extract from the product.  
Platform admins use Proton because they can: 
- create standard and versioned templates to better support their developers 
- create guard-rails in the SDLC that guarantee governance and compliance 
- have a consolidated view of the stacks deployed in the organization (including their versions)
- and more... 
Developers use Proton because they can:
- deploy their application in self-service 
- deploy their applications without being experts in AWS services
- avoid to reinvent the wheel and re-use best practices (for security, availability, cost optimization) codified by the platform teams in the templates 
- and more... 


