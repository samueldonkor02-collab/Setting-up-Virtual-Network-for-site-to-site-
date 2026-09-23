# Setting-up-Virtual-Network-for-site-to-site-
A two subnet Azure VNet simulating two office sites, with a Windows Firewall rule added to enable ping connectivity between the site VMs.


AZURE VIRTUAL NETWORK SEGMENTATION AND CROSS SUBNET CONNECTIVITY
(Multiple Subnets, VM Deployment Across Sites, Windows Firewall ICMP Rule)

Tags: Microsoft Azure, Virtual Networks, Subnetting, Windows Server 2025, Windows Defender Firewall, ICMP, Network Segmentation

OVERVIEW
For this lab I built out a single Azure virtual network split into multiple subnets to simulate two separate office sites sharing one network, then deployed a VM into each site subnet and tried to get them talking to each other. This ended up being less about spinning up VMs and more about understanding what actually happens at the network layer when two machines sit on different subnets inside the same VNet, and why a simple ping between them does not just work out of the box.

OBJECTIVE
Design and build a VNet with a proper address space split into multiple subnets, deploy one Windows Server VM into each of two site subnets, confirm each VM's network configuration from the inside using ipconfig, test connectivity between the two VMs across subnets, and fix any blockers standing in the way of that connectivity.

ENVIRONMENT
Cloud platform: Microsoft Azure (Free Trial subscription)
Resource group: v net production
Virtual network name: v net production
Address space: 192.168.0.0/16 (65,536 addresses total)
Subnets:
  default        192.168.0.0/24    (256 addresses)
  site A LA      192.168.10.0/24   (256 addresses)
  site B LA      192.168.20.0/24   (256 addresses)

VM 1: site1
  Subnet: site A LA
  Private IP: 192.168.10.4
  Public IP: 20.228.94.168
  OS: Windows Server 2025 Datacenter
  VM generation: V2, architecture x64
  Size: Standard D2s v3 (2 vCPUs, 8 GiB RAM)

VM 2: site2
  Subnet: site B LA
  Private IP: 192.168.20.4
  Public IP: 52.190.249.60
  OS: Windows Server 2025 Datacenter
  VM generation: V2, architecture x64
  Size: Standard D2s v3 (2 vCPUs, 8 GiB RAM)

Client device: ASUS Vivobook, used to RDP into both VMs

TOOLS I USED
Azure Portal            Web based console used to build the VNet, define subnets, and provision both VMs
Remote Desktop Protocol Used to connect into site1 and site2 to run commands directly on each machine
PowerShell (ipconfig)   Confirmed each VM's private IP, subnet mask, and default gateway from inside the OS
netsh advfirewall       Used to add an explicit inbound rule allowing ICMPv4 echo requests through Windows Defender Firewall
ping                    Used to test actual reachability between the two VMs once the firewall rule was in place
Snipping Tool           Captured screenshots of the terminal output for documentation

WHAT I DID

Building a segmented virtual network
Instead of leaving everything on one flat default subnet, I built the v net production VNet with a 192.168.0.0/16 address space and then carved out two additional subnets on top of the default one: site A LA at 192.168.10.0/24 and site B LA at 192.168.20.0/24. The idea was to model two separate site networks living under one VNet, which is a pretty common real world pattern for a small business with more than one office or department that still needs to share some infrastructure.

Deploying a VM into each site subnet
I created site1 inside the site A LA subnet and site2 inside the site B LA subnet. Azure assigned site1 the private IP 192.168.10.4 with a public IP of 20.228.94.168, and site2 came back with a private IP of 192.168.20.4 and a public IP of 52.190.249.60. Right after site2 finished provisioning, its agent status showed as Not Ready in the Azure Portal, which is normal immediately after creation while the VM agent extension is still starting up in the background.

Confirming network configuration from inside each VM
I RDP'd into both machines separately and ran ipconfig on each one to verify what Azure had assigned. On site1, ipconfig showed the IPv4 address 192.168.10.4, subnet mask 255.255.255.0, and default gateway 192.168.10.1. On site2, ipconfig showed 192.168.20.4, the same subnet mask, and a default gateway of 192.168.20.1. Both machines also picked up an internal cloudapp.net DNS suffix from Azure automatically.

Hitting a wall on cross subnet connectivity
With both VMs confirmed to be on their correct subnets, the next step was proving they could actually reach each other across the VNet. A straight ping from one VM to the other would not succeed on its own, because Windows Server ships with Windows Defender Firewall blocking inbound ICMP echo requests by default. Without an explicit allow rule, the receiving VM simply drops the ping request before it ever gets a reply.

Opening up ICMP with an explicit firewall rule
On site1, I ran the following command in an elevated PowerShell session:
  netsh advfirewall firewall add rule name="Allow ICMPv4 In" protocol=icmpv4:8,any dir=in action=allow
This adds a rule that lets ICMPv4 type 8 (echo request) traffic through inbound, which is exactly what a ping needs. I did the same thing on site2 so that both machines could receive and respond to ping requests from each other.

Verifying two way connectivity
After the firewall rule was in place on both VMs, I pinged 192.168.20.4 from site1 and got four replies back with zero percent packet loss and an average round trip time of around one millisecond. Then I went to site2 and pinged 192.168.10.4, which also came back clean with four replies, zero loss, and about a one millisecond average. That confirmed traffic was flowing correctly in both directions between two VMs sitting on completely different subnets inside the same virtual network.

WHAT'S IN THIS REPO
azure vnet subnetting lab/
  README.txt                                   This file
  screenshots/
    01 vnet create ip addresses subnets.png     VNet address space and the three configured subnets
    02 site2 overview networking.png            site2 VM overview showing private/public IP and subnet
    03 site1 overview networking.png            site1 VM overview showing private/public IP and subnet
    04 site1 ipconfig ping site2.png            site1 terminal, ipconfig, firewall rule, ping to site2
    05 site2 ipconfig ping site1.png            site2 terminal, ipconfig, firewall rule, ping to site1

SKILLS I PICKED UP
Planning an address space and splitting it into multiple subnets that model separate sites instead of dropping everything into one flat network.
Understanding that a VNet with multiple subnets does not automatically mean two machines on different subnets can freely ping each other, since the host level firewall still has the final say.
Using netsh advfirewall to add a targeted inbound rule instead of just disabling the firewall entirely to make something work.
Verifying network configuration and reachability from inside the OS with ipconfig and ping rather than only trusting what the Azure Portal shows.
Working across two separate RDP sessions at once to test connectivity from both sides rather than assuming a one direction test proves everything is fine.

HOW THIS APPLIES IN THE REAL WORLD
This mirrors a very common setup where a company has more than one physical site or department, each mapped to its own subnet, all living inside the same VNet so they can eventually share resources. The security relevant part is that Azure routing traffic between subnets is not the same as a host actually accepting that traffic, and ICMP being blocked by default is a deliberate hardening choice in Windows Server, not a bug. In a real environment the next step past this lab would be using Network Security Groups at the subnet level to control exactly which ports and protocols are allowed to cross from one site subnet to another, rather than opening ICMP broadly on every host the way I did here just to prove connectivity.

WHERE I'M COMING FROM
I'm making the jump into cybersecurity from a background in healthcare. I'm currently studying for CompTIA Security+ and building labs like this one to get real hands on reps with networking fundamentals, since understanding how traffic actually moves (or gets blocked) between subnets feels like a prerequisite for understanding how to segment and secure a real network.

WHAT I WANT TO LEARN NEXT
Applying Network Security Groups at the subnet level to control site A LA to site B LA traffic instead of relying on host based firewall rules alone.
Setting up VNet peering to connect two separate VNets rather than two subnets inside one VNet.
Looking at what other protocols besides ICMP are blocked by default on Windows Server and deciding which ones actually need to be opened for a given use case.
Capturing traffic between the two VMs to see the ping requests and replies at the packet level instead of only trusting the command line output.

LIMITATIONS AND WHAT I'D DO DIFFERENTLY IN PRODUCTION
I opened ICMP broadly on both VMs (protocol=icmpv4:8,any) instead of scoping the rule down to a specific source subnet, which is fine for a lab but too permissive for production.
No Network Security Group was configured at the subnet level to control what type of traffic is allowed to move between site A LA and site B LA, so right now anything routable between them could technically pass once a given port or protocol is opened on the host.
Both VMs were still reachable over RDP from any public IP address, which is the same testing only exposure I called out in the previous IIS lab and would need to be locked down before this setup went anywhere near production.

REFERENCES
Microsoft Azure Virtual Network Documentation: https://learn.microsoft.com/en-us/azure/virtual-network/
Windows Defender Firewall with Advanced Security Documentation: https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/
CompTIA Security+ (SY0-701) Exam Objectives: https://www.comptia.org/certifications/security
