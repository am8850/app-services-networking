# Azure App Services Networking and Security

Many customers coming to Azure ask the question, "is our design secure?" Security is a big and important topic, but a good answer has to do with meeting the security compliance requirements of your organization. Understanding which model will meet these requirements is key to being in compliance.

The purpose of this post is to be able to quickly compare the different secuirty consideration for different App Services networking models. For simplicity, the post is based on a two-tier architecture application, but this can quickly be extended to 3-tier, microservices, and other more complex architectures involving other Azure Services as the same concepts apply.

## App Service Overview

## App Service Networking Features

- Access Restrictions: IP and Service Tags rules that can be applied as inbound rules to the App Service
- VNET Integration: A feature of App Services that allows it to connect to private services via a backend subnet
- Route ALL Switch: In App Service VNET integration this switch will tell the networking if all traffic should exit the App Service through the Azure backbone
- UDRs: A UDR can be applied to traffic from the backed subnet to for example force traffic to egress from a firewall
- NAT Gateway: A NAT gateway can be applied to the outbound IPs
- ASE Subnet Deployment

## Resiliency

- Leverage scaling including up and down and in and out
- Keep at least two instances
- Leverage multi-zonal deployments
- Deploy with services with IaC
- Leverage DevOps CI/CD piplelines
- Monitoring

## Cost

- Right sizing
- Scaling
- Monitoring

## App Service Design - Where is that Public IP?

- An App Service has a reverse proxy into the instances
- The reverse proxy exposes a public IP

```mermaid
graph LR;
    Z((Public IP))-->A;
    A(Reverse<br/>Proxy)-->B(Instance 1);
    A-->C(Instance 2);
    A-->D(Instance 3);
    classDef internet fill:#007FFF,color:white;
    class Z internet;
    classDef proxy fill:magenta,color:white;
    class A proxy;
    classDef instances fill:purple,color:white;
    class B,C,D instances;    
```
## Other security best practices for App Services and Azure SQL
- Enable identity
- For ASE, be careful not to block certain ports (via NSGs) which are required for the service to operate correctly
- Enable logging and monitoring
- Enable encryption at rest

## App Services solution deployments in order or improved security

### App Service with IP Filter & Azure SQL with Service Tags

```mermaid
graph LR;
  A((Internet))-- IP Filter -->a1;
  subgraph "App Service Plan"
    a1(Web App)
  end
  a1-- Service Tags -->C(Azure<br/>SQL);
  classDef internet fill:#007FFF,color:white;
  classDef unsafe fill:#ff3333,color:white;
  classDef semisafe fill:darkorange,color:black;
  classDef safe fill:darkgreen,color:white;
  class A internet;
  class a1 unsafe;
  class C safe;
```

Azure Services:
- App Service plan
- App Service Web App
- Azure SQL

Security at this level:
- No WAF (recommended)
- TLS enforced and custom certificate can be added to the Web App
- Traffic into the web app can be limited to one IP (i.e. the corporate firewall IP). Otherwise, it is encrypted but open.
- Traffic from App to Data can only come from app services by setting the ServiceTags in the Database firewall settings.
- All traffic traverses the internet

Suitable scenarios:
- Dev, Test, POC, Personal Project

### Application Gateway, App Service with Service Tags, and Azure SQL with Firewall with Service Tags

```mermaid
graph LR;
  A((Internet))-->B((Public IP));
  B-->a2;
  subgraph "Subnet"
    a2(AppGw<br/>WAF);
  end
  a2--Subnet<br/>Restriction-->a1;
  subgraph "App Service Plan"
    a1(Web App)
  end
  a1-- Service Tags -->E(Azure<br/>SQL);
  classDef internet fill:#007FFF,color:white;
  classDef unsafe fill:#ff3333,color:white;
  classDef semisafe fill:darkorange,color:black;
  classDef safe fill:darkgreen,color:white;
  class A internet;
  class E semisafe;
  class C,a1,a2,E safe;
```

Azure Services:
- Public IP
- VNET
- Application Gateway in WAF mode deployed to a subnet
- App Service plan
- App Service Web App
- Azure SQL

Security at this level:
- Public IP (DDOS protection)
- WAF protection
- TLS enforced and custom certificate can be added to the Web App
- Application Gateway can do SSL offloading but can also handle end-to-end encryption
- Traffic into the web app can only come from application gateway
- Traffic from App to Data can only come from app services via database firewall using ServiceTags
- All traffic traverses the internet

Suitable scenarios:
- QA, UAT, PROD


### Front Door Standard, App Service with Service Tags and VNET integration, and Azure SQL with Private Endpoint

```mermaid
graph LR;
  A((Internet))-->B(Front Door<br/>WAF);
  B-- Service Tags -->a1;
  subgraph "App Service Plan"
    a1(Web App);
  end; 
  a1-->a3;
  subgraph "Backend<br/>Subnet"
    a3(VNET<br/>Integration);
  end;
  a3-->a2;
  subgraph "Private Endpoint<br/>Subnet"
    a2(Private<br/>Endpoint);
  end;
  a2-->E(Azure<br/>SQL);
  classDef internet fill:#007FFF,color:white;
  classDef unsafe fill:#ff3333,color:white;
  classDef semisafe fill:darkorange,color:black;
  classDef safe fill:darkgreen,color:white;
  class A internet;
  class B semisafe;
  class a1,a2,E safe;
```

Azure Services:
- Azure FrontDoor Standard Plan
- VNET
- NSG
- App Service plan
- App Service Web App with VNET integration
  - Note: VNET integration requires a backend subnet in a VNET 
- Private DNS Zone (Azure SQL)
- Private Endpoint deployed to a VNET subnet
- Private Endpoint for Azure SQL
- Azure SQL under Private Endpoint

Security at this level:
- WAF protection
- TLS enforced and cad add custom certificate to Azure Froont Door for SSL offloading or have end-to-end encryption
- Traffic into the web app can only come from Azure FrontDoor
- Traffic from App to Data can only come from app services via the backend subnet into the Private Endpoint into Azure SQL
- Traffic between the Web App and Data traverses the Azure backbone
- Traffic from the FrontDoor to the App goes over the internet

### Front Door Premium, App Service under Private Endpoint and VNET integration, and Azure SQL with Private Endpoint

```mermaid
graph LR;
  A((Internet))-->B(Front Door<br/>WAF);
  B-->a1;
  subgraph "Pe Subnet"
  a1(Private<br/>Endpoint);
  end;
  a1-->a2;
  subgraph "App Service Plan"
  a2(Web App);
  end;
  a2-->a3;
  subgraph "Backend<br/>Subnet"
  a3(VNET<br/>Integration);
  end;  
  a3-->a4;
  subgraph "Pe Subnet"
  a4(Private<br/>Endpoint);
  end;
  a4-->F(Azure<br/>SQL);
  classDef internet fill:#007FFF,color:white;
  classDef unsafe fill:#ff3333,color:white;
  classDef semisafe fill:darkorange,color:black;
  classDef safe fill:darkgreen,color:white;
  class A internet;
  class B semisafe;
  class a1,a2,a4,F safe;
```

Azure Services:
- Azure FrontDoor Premium Plan
- VNET
- NSG
- App Service plan
- App Service Web App under Private Endpoint with VNET integration
  - Note: VNET integration requires a backend subnet in a VNET 
- Private DNS Zones (Azure SQL and Websites)
- Private Endpoints deployed to a VNET subnet
- Private Endpoint for Azure SQL
- Azure SQL under Private Endpint

Security at this level:
- WAF protection
- TLS enforced and cad add custom certificate to Azure Froont Door for SSL offloading or end-to-end encryption
- Traffic into the web app can only come from Azure FrontDoor and the Azure backbone
- Traffic from App to Data can only come from app services via the backend subnet into the Private Endpoint into Azure SQL
- All traffic flows inside the Microsoft backbone

### App Service Environments (ASE v3) and Azure SQL with Private Endpoint

```mermaid
graph LR;
  A((Internet))-->B((Public IP));
  B-->a1;
  subgraph "AppGw Subnet"
    a1(AppGw<br/>WAF);
  end;
  a1-->a2;
  subgraph "ASE Subnet"
    a2(ASE v3);
  end;
  a2-->a3;
  subgraph "PE Subnet"
    a3(Private<br/>Endpoint);
  end;
  a3-->F(Azure SQL);
  classDef internet fill:#007FFF,color:white;
  classDef unsafe fill:#ff3333,color:white;
  classDef semisafe fill:darkorange,color:black;
  classDef safe fill:darkgreen,color:white;
  class A internet;
  class a1,a2,a3,F safe;    
```

Azure Services:
- VNET
- NSG
- Public IP
- Application Gateway in WAF mode
- App Service Isolated plan
- App Service Web App with VNET integration deployed into subnets
- Private DNS Zone
- Private Endpoint for Azure SQL
- Azure SQL

Security at this level:
- Public IP (DDOS protection)
- - WAF protection
- TLS enforced, cad add custom certificate to Application Gateway, and Application Gateway can do SSL offloading
- App Service Environment deployed to VNET subnet obtaining a private IP
- SQL MI deployed to a VNET subnet obtaining a private IP
- Traffic into the web app can only come from application gateway via the private IP
- Traffic from the Web App to Azure SQL can only come from app services via the backend subnet and the Private Endpoint
- All traffic flows inside the Microsoft backbone


### App Service Environments (ASE v3) and SQL MI

```mermaid
graph LR;
  A((Internet))-->B((Public IP));
  B-->a1;
  subgraph "AppGw Subnet"
  a1(AppGw<br/>WAF);
  end;
  a1-->a2;
  subgraph "ASE Subnet"
    a2(ASE v3);
  end;
  a2-->a3;
  subgraph "SQL Subnet"
    a3(SQL MI);
  end;  
  classDef internet fill:#007FFF,color:white;
  classDef unsafe fill:#ff3333,color:white;
  classDef semisafe fill:darkorange,color:black;
  classDef safe fill:darkgreen,color:white;
  class A internet;  
  class a1,a2,a3 safe;    
```

Azure Services:
- VNET
- NSG
- Public IP
- Application Gateway in WAF mode
- App Service Isolated plan
- App Service Web App with VNET integration deployed into subnets
- Azure SQL MI deployed to a subnet

Security at this level:
- Public IP (DDOS protection)
- AppGw provides SSL offloading WAF protection
- TLS enforced, cad add custom certificate to Application Gateway
- App Service Environment deployed to VNET subnet obtaining a private IP
- SQL MI deployed to a VNET subnet obtaining a private IP
- Traffic into the web app can only come from application gateway via the private IP
- Traffic from the Web App to Azure SQL MI can only come from app services via the backend subnet
- All traffic flows inside the Microsoft backbone and there are no public IPs

### References

- [Front Door Premium and App Service](https://learn.microsoft.com/en-us/azure/frontdoor/standard-premium/how-to-enable-private-link-web-app)
- [Azure SQL-Prinvate Endpoint](https://learn.microsoft.com/en-us/azure/private-link/tutorial-private-endpoint-sql-portal)
- [ASE v3 Networking](https://learn.microsoft.com/en-us/azure/app-service/environment/networking)
