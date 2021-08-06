# VPC Setup
## Notes
`10.211.0.0` was used as a baseline for this, replace it with your own private IP.

Please note also that to keep things consistent we will be using:
- `10.211.1.0` to reference the public subnet
- `10.211.2.0` to reference the private subnet
- `10.211.3.0` to reference the Bastion's subnet 

## Steps for creation
### Create a new VPC, the private IP address as the IPv4 CIDR block
1. Alternatively, click on services, and scroll down to the heading `Networking and Content Delivery` where `VPC` should be under it.
2. To create a new VPC, click the `Create VPC` in the top right
3. Give the VPC a suitable name using the similar naming convention used in the past e.g., `eng89_yourname_vpc`
4. For the IPv4 CIDR block - use the IP - `10.211.0.0/16`
5. We can leave everything the same but optionally, a description tag can be added to tell others the purpose of the VPC

### Create an internet gateway
1. Name and Add description
2. Select VPC
3. Add inbound and outbound rules
4. Optional name tag
5. Do the same for all servers

### Create a route table

    |   Destination    |   Target                    |   Status  |   Propagated  |
    |------------------|-----------------------------|-----------|---------------|
    |   `10.211.0.0/16`  |   local                     |   Active  |   No          |
    |   `0.0.0.0/0`      |   Your internet gateway id  |   Active  |   No          |
4. Create 3 ACLs
    1. Public:
        1. Inbound:

        | Rule number | Type       | Protocol | Port range   | Source       | Allow/Deny |
        |-------------|------------|----------|--------------|--------------|------------|
        | 100         | HTTP (80)  | TCP (6)  | `80`           | `0.0.0.0/0`    | Allow      |
        | 115         | SSH (22)   | TCP (6)  | `22`           | `[your ip]/32` | Allow      |
        | 120         | Custom TCP | TCP (6)  | `1024 - 65535` | `0.0.0.0/0`    | Allow      |    
		2. Outbound:
    
        | Rule number | Type       | Protocol | Port range   | Destination   | Allow/Deny |
        |-------------|------------|----------|--------------|---------------|------------|
        | 100         | HTTP (80)  | TCP (6)  | `80`           | `0.0.0.0/0`     | Allow      |
        | 110         | Custom TCP | TCP (6)  | `27017`        | `10.211.2.0/24` | Allow      |
        | 120         | Custom TCP | TCP (6)  | `1024 - 65535` | `0.0.0.0/0`     | Allow      |
    2. Private: 
        1. Inbound:
        
        | Rule number | Type       | Protocol | Port range   | Source        | Allow/Deny |
        |-------------|------------|----------|--------------|---------------|------------|
        | 100         | Custom TCP | TCP (6)  | `27017`        | `10.211.1.0/24` | Allow      |
        | 110         | Custom TCP | TCP (6)  | `1024 - 65535` | `0.0.0.0/0`     | Allow      |
        | 120         | SSH (22)   | TCP (6)  | `22`           | `10.211.3.0/24` | Allow      |
        2. Outbound:
        
        | Rule number | Type       | Protocol | Port range   | Destination   | Allow/Deny |
        |-------------|------------|----------|--------------|---------------|------------|
        | 100         | HTTP (80)  | TCP (6)  | `80`           | `0.0.0.0/0`     | Allow      |
        | 110         | Custom TCP | TCP (6)  | `1024 - 65535` | `10.211.1.0/24` | Allow      |
        | 120         | Custom TCP | TCP (6)  | `1024 - 65535` | `10.211.3.0/24` | Allow      |
    3. Bastion:
        1. Inbound:
        
        | Rule number | Type       | Protocol | Port range   | Source        | Allow/Deny |
        |-------------|------------|----------|--------------|---------------|------------|
        | 100         | SSH (22)   | TCP (6)  | `22`           | `[your ip]/32`     | Allow      |
        | 110         | Custom TCP | TCP (6)  | `1024 - 65535` | `10.212.2.0/24` | Allow      |
        2. Outbound:
        
        | Rule number | Type       | Protocol | Port range   | Destination   | Allow/Deny |
        |-------------|------------|----------|--------------|---------------|------------|
        | 100         | SSH (22)   | TCP (6)  | `22`           | `10.211.2.0/24` | Allow      |
        | 110         | Custom TCP | TCP (6)  | `1024 - 65535` | `0.0.0.0/0`     | Allow      |
5. Create 3 subnets in the eu-west-1a availability zone:
    1. Public: `10.211.1.0`
        1. Assign to public acl
        2. Assign to your route table
    2. Private: `10.211.2.0`
        1. Assign to private acl
    3. Bastion: `10.211.3.0`
        1. Assign to public acl
        2. Assign to your route table
6. Create 3 security groups:
    1. app:
        1. Inbound:
        | Type       | Protocol | Port range   | Source        | Description |
        |------------|----------|--------------|---------------|------------|
        | HTTP       | TCP (6)  | `80`         | `0.0.0.0/0`   | Allows users to connect      |
        | SSH        | TCP (6)  | `22`         | `[your ip]/32` |  Allows us to SSH in  |
        2. Outbound:

        | Type       | Protocol | Port range   | Source        | Description |
        |------------|----------|--------------|---------------|------------|
        | All traffic| TCP (6)  | All          | `0.0.0.0/0`   | Allows all traffic leaving      |
        
    2. Bastion
        1. Inbound:
        | Type       | Protocol | Port range   | Source        | Description |
        |------------|----------|--------------|---------------|------------|
        | SSH        | TCP (6)  | `22`         | `[your ip]/32`|  We only want the ssh set up since this talks between us and the db  |

        2. Outbound:

        | Type       | Protocol | Port range   | Source        | Description |
        |------------|----------|--------------|---------------|------------|
        | All traffic| TCP (6)  | All          | `0.0.0.0/0`   | Allows all traffic leaving      |

    3. db
        1. Inbound: link to your app and you db security groups on port `27017` and `22` respectively
        2. Outbound: Leave as default
7. Head on over to EC2 and spin up 3 new instances:
    1. App
        - choose app AMI
        - Select VPC for - Network
        - Select app subnet
        - Public IP - Enable
        - Add sutable name tag
        - Pick app SG
        - Launch using eng89_devops key pair
    2. DB
        - choose db AMI
        - Select VPC for - Network
        - Select db subnet
        - Public IP - Disable
        - Add suitable name tag
        - Pick db SG
        - Launch using eng89_devops key pair
    3. Bastion
        - choose `Ubuntu Server 16.04 LTS (HVM), SSD Volume Type`
        - Select VPC for - Network
        - Select bastion subnet
        - Public IP - Enable
        - Add suitable name tag
        - Pick bastion SG
        - Launch using eng89_devops key pair
8. (For each step, select the matching VPC and corresponding subnet & security group for each instance)
9. (Make sure that for app and Bastion you allow for a public IP to be automatically assigned)

## Accessing the db from bastion server
In order to access the DB you will have to go via your Bastion instance. Here's how:
- From local `.ssh/` directory `scp -i eng89_devops.pem eng89_devops.pem ubuntu@bastion.IP.address:~/`
- ssh to bastion server
- `mv eng89_devops.pem .ssh/`
- `cd .ssh`
- `chmod 400 eng89_devops.pem` to make it invisable
- `ssh -i "eng89_devops.pem" ubuntu@db.private.IP`

### To solve sudo not working, add IP
- `sudo nano /etc/hosts`
- add private IP in line 2, for example:`10.211.1.13 ip-10-211-1-13`

## Quick reminder:
1. Inside db:
    - `sudo nano /etc/mongod.conf` change to `bindIp: 0.0.0.0`
    - `sudo systemctl restart mongod`
2. Inside app:
    - `export DB_HOST=mongodb://[db private ip]:27017/posts >> ~/.profile`
    - Change `proxy_pass` ip address to [public app ip] inside `/etc/nginx/sites-available/default`
    - `sudo nginx -t`
    - `sudo systemctl restart nginx`
    - Run seeds.js (`app/seeds/`)
    - Run `npm start` in `app/` directory
