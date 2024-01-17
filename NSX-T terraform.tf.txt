#********************************************************************************

#Creating Terraform providers for VMware NSX-T

#********************************************************************************

 

provider "nsxt" {

  host                                   =       "<provide your NSX-T Manager IP or FQDN>"

  username                         =       "<NSX-T Manager username>"

  password                          =       "<NSX-T Manager username>"

  allow_unverified_ssl       =        false

  max_retries                      =        10

  retry_min_delay              =        500

  retry_max_delay              =        5000

  retry_on_status_codes   =        [429]

}

 

# Creating Data Sources for existing NSX-T Overlay Transport Zones

 

Data "nsxt_policy_tranpsort_zone" "overlay_tz" {

     display_name = "Overlay-TZ"

}

 

# Creating Data Sources for existing NSX-T Edge Cluster

 

Data "nsxt_policy_edge_cluster "edge_cluster" {

     display_name = "EdgeCluster-01"

}

 

# Creating Data Sources for existing NSX-T T0 Gateway

 

Data "nsxt_policy_t0_gateway" "t0_gateway" {

     display_name = "Terraform-Tier-0"

 

#Creating resources for new Tier-1 gateway

 

resource "nsxt_policy_tier1_gateway" "tier1_gateway" {

   display_name                          =       "Terraform-Tier-0"

   description                               =       "Tier1 Created by Terraform"

   edge_cluster_path                 =         data.nsxt_policy_edge_cluster.edge_cluster.path

   enable_firewall                       =        "true"

   tier0_path                                =         data.nsxt_policy_tier0_gateway.t0_gateway.path

   route_advertisement_types =         ["TIER1_STATIC_ROUTES", "TIER1_CONNECTED"]

}

 

# Creating a Web-Segment 10.1.1.0/24

 

resource "nsxt_policy_Segment" "web1" {

   display_name                          =          "Web-Segment"

   connectivity_path                   =         data.nsxt_policy_tier1_gateway.tier1_gateway.path

   transport_zone_path             =          data.nsxt_policy_tranpsort_zone.overlay_tz.path

    subnets {

                  cidr = "10.1.1.0/24"

                  }

 }

 

# Creating an App-Segment 10.1.2.0/24

 

resource "nsxt_policy_Segment" "app1" {

   display_name                          =          "App-Segment"

   connectivity_path                   =         data.nsxt_policy_tier1_gateway.tier1_gateway.path

   transport_zone_path             =          data.nsxt_policy_tranpsort_zone.overlay_tz.path

    subnets {

                  cidr = "10.1.2.0/24"

                  }

 }

 

# Creating a DB-Segment 10.1.3.0/24

 

resource "nsxt_policy_Segment" "DB1" {

   display_name                          =          "DB-Segment"

   connectivity_path                   =         data.nsxt_policy_tier1_gateway.tier1_gateway.path

   transport_zone_path             =          data.nsxt_policy_tranpsort_zone.overlay_tz.path

    subnets {

                  cidr = "10.1.3.0/24"

                  }

 }

 

# Creating Security Group for Web

 

resource "nsxt_policy_group" "group1" {

 display_name                         =       "Web-Group"

 description                              =       "Web group created by Terraform"

 criteria {

     condition  {

           key                                 =        "Name"

           member_type              =        "VirtualMachine"

           operator                        =        "STARTSWITH"

           value                              =        "web"

            }

        }

}

 

# Creating Security Group for App

 

resource "nsxt_policy_group" "group2" {

 display_name                         =       "App-Group"

 description                              =       "App group created by Terraform"

 criteria {

     condition  {

           key                                 =        "Name"

           member_type              =        "VirtualMachine"

           operator                        =        "STARTSWITH"

           value                              =        "app"

            }

        }

}

 

# Creating Security Group for DB

 

resource "nsxt_policy_group" "group3" {

 display_name                         =       "DB-Group"

 description                              =       "DB group created by Terraform"

 criteria {

     condition  {

           key                                 =        "Name"

           member_type              =        "VirtualMachine"

           operator                        =        "STARTSWITH"

           value                              =        "db"

            }

        }

}

 

# Creating data sources for existing HTTPS, and MYSQL Services

 

data "nsxt_policy_service" "http" {

  display_name                        = "HTTPS"

}

 

data "nsxt_policy_service" "mysql" {

  display_name                        = "MYSQL"

}

 

# Creating resources for new TCP-8443 Service

 

resource "nsxt_policy_service" "service_8443" {

  display_name                          =         "TCP-8443"

  description                               =        "This Service is created by Terraform"

  l4_set_port_entry {

   display_name                         =         "TCP-8443"

   description                              =         "TCP 8443 port"

    protocol                                  =         "TCP"

    destination_ports                 =        ["8443"]         

     }

}

 

# Creating distributed firewall rules using Terraform

 

resource "nsxt_policy_security_policy" "policy_3_tier" {

  display_name           =  "3-Tier-policy"

  description                =   "Security Policy created by Terraform"

  locked                        =   true

  stateful                       =    true

  tcp_strict                    =    true

  category                      =    "Application"

   scope                          =    [nsxt_policy_group.group1.path, nsxt_policy_group.group2.path, nsxt_policy_group.group1.path]

   rule {

        display_name       =   "Allow All traffic to Web-VMs"

        destination_groups  =   [nsxt_policy_group.group1.path]

        services                       =   [data.nsxt_policy_service.https.path]

        action                          =   "Allow"

        logged                         =     true

        }

  rule {

        display_name       =   "TCP-8443 traffic from Web to App"

        source_groups          = [nsxt_policy_group.group1.path]

        destination_groups  =   [nsxt_policy_group.group2.path]

        services                       =   [data.nsxt_policy_service.service_8443.path]

        action                          =   "Allow"

        logged                         =     true

        }

 rule {

        display_name       =   "TCP-8443 traffic from Web to App"

        source_groups          = [nsxt_policy_group.group2.path]

        destination_groups  =   [nsxt_policy_group.group3.path]

        services                       =   [data.nsxt_policy_service.mysql.path]

        action                          =   "Allow"

        logged                         =     true

        }

 rule {

        display_name       =   "Drop all other traffic"

        action                     =    "REJECT"

        logged                    =     true

        }

}

 

# Creating Gateway firewall rules using Terraform

resource "nsxt_policy_gateway_policy" "gateway_3_policy"{

 display_name           =         "Gateway-3-policy"

 description                =          "Gateway Policy created by Terraform"

 locked                        =.          true

 category                    =           "LocalGatewayRules"

 stateful                      =            true

 tcp_strict                   =            false

 sequence_number  =            1

rule {

        display_name       =   "HTTPS traffic to Web Servers"

        destination_groups  =   [nsxt_policy_group.group1.path]

        services                       =   [data.nsxt_policy_service.https.path]

        action                          =   "ALLOW"

        logged                         =     true

        scope                           =     [nsxt_policy_tier1_gateway.tier1_gateway.path]

        }

rule {

        display_name       =   "Drop all other traffic"

        action                          =   "REJECT"

        logged                         =     true

        scope                           =     [nsxt_policy_tier1_gateway.tier1_gateway.path]

        }

}

 

