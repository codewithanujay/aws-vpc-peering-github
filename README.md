# AWS VPC Peering with Private EC2 SSH Connectivity

A hands-on AWS networking project demonstrating private communication between two VPCs using VPC Peering, route tables, security groups, a private EC2 instance, AWS Systems Manager Session Manager, and SSH key authentication.

> **Security:** This repository intentionally contains no PEM files, SSH private keys, passwords, passphrases, AWS access keys, session tokens, or raw terminal screenshots. Never commit secrets. If any password/passphrase was exposed in a screenshot or chat, rotate it before reusing it.

## Architecture

- **Region:** `ap-south-1` (Mumbai)
- **VPC A:** `10.10.0.0/16`
  - Public subnet: `10.10.1.0/24`
  - EC2 Server A private IP: `10.10.1.147`
- **VPC B:** `10.20.0.0/16`
  - Private subnet: `10.20.2.0/24`
  - EC2 Server B private IP: `10.20.2.71`
- **VPC Peering:** Active; routes configured in both VPC route tables.
- **Private access:** Server B is reachable through Systems Manager Session Manager.
- **Verified connectivity:** ICMP ping succeeds in both directions; TCP port 22 is reachable from A to B; SSH login from A to B succeeded after adding Server A's public key to `/home/ec2-user/.ssh/authorized_keys` on Server B.

See [`diagrams/architecture.svg`](diagrams/architecture.svg).

## Resources

| Resource | Configuration |
|---|---|
| VPC A | `10.10.0.0/16` |
| VPC A public subnet | `10.10.1.0/24` |
| VPC B | `10.20.0.0/16` |
| VPC B private subnet | `10.20.2.0/24` |
| Server A | `10.10.1.147` |
| Server B | `10.20.2.71` |
| Peering connection | Active, routes on both sides |
| Server B access | AWS Systems Manager Session Manager |

Resource IDs and public IPs are intentionally omitted from the reusable guide because they are account-specific and can change. Add your own values locally when following the guide.

## How it works

1. Create two VPCs with non-overlapping CIDR ranges.
2. Create the required subnets and launch EC2 instances.
3. Create a VPC Peering connection between VPC A and VPC B and accept it.
4. Add routes for the opposite VPC CIDR to each VPC's route table.
5. Configure security groups to permit only the required traffic from the peer VPC CIDR or relevant instance/subnet.
6. Attach an IAM role with `AmazonSSMManagedInstanceCore` to the private instance and connect using Session Manager.
7. Test private IP connectivity in both directions.
8. Configure SSH public-key authentication if direct SSH from Server A to Server B is required.

## Route table entries

Add these routes to the route tables associated with the subnets that need cross-VPC connectivity:

| Route table | Destination | Target |
|---|---|---|
| VPC A route table | `10.20.0.0/16` | VPC Peering connection ID (`pcx-...`) |
| VPC B private route table | `10.10.0.0/16` | Same VPC Peering connection ID (`pcx-...`) |

Keep each VPC's local route. VPC Peering does not automatically add routes.

## Security group guidance

Use least privilege. Example rules for this lab:

- **Server A security group:** allow SSH from your trusted public IP (`your-ip/32`). For ping testing from VPC B, temporarily allow ICMP Echo Request from `10.20.0.0/16` if needed.
- **Server B security group:** allow SSH TCP/22 from Server A's private IP (`10.10.1.147/32`) or the narrowest appropriate source. For ping testing from VPC A, temporarily allow ICMP from `10.10.0.0/16`.
- Do not open SSH to `0.0.0.0/0`.
- Remove temporary ICMP rules after testing if they are no longer required.

## SSH from Server A to Server B

On Server A, create a dedicated key pair for this connection (the private key stays on Server A):

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ewb-server-b-key
chmod 600 ~/.ssh/ewb-server-b-key
```

Display only the **public** key:

```bash
cat ~/.ssh/ewb-server-b-key.pub
```

Using Session Manager on Server B, add that public key to the correct Linux user's `authorized_keys` file. For Amazon Linux, the login user is commonly `ec2-user`; confirm the correct account first. For example, when connected as an SSM user and using `sudo`:

```bash
sudo install -d -m 700 -o ec2-user -g ec2-user /home/ec2-user/.ssh
sudo touch /home/ec2-user/.ssh/authorized_keys
sudo chown ec2-user:ec2-user /home/ec2-user/.ssh/authorized_keys
sudo chmod 600 /home/ec2-user/.ssh/authorized_keys
```

Append the public key (replace the example text with the full public key from Server A):

```bash
printf '%s\n' 'ssh-ed25519 AAAA... your-key-comment' | sudo tee -a /home/ec2-user/.ssh/authorized_keys >/dev/null
sudo chown ec2-user:ec2-user /home/ec2-user/.ssh/authorized_keys
sudo chmod 600 /home/ec2-user/.ssh/authorized_keys
```

Then, on Server A:

```bash
ssh -i ~/.ssh/ewb-server-b-key -o IdentitiesOnly=yes ec2-user@10.20.2.71
```

The key file must remain private and must never be uploaded to GitHub. If you set a passphrase, enter it when prompted; do not paste it into chat or a repository.

## Verification commands

Run from Server A or Server B as indicated:

```bash
# From Server A: test Server B private IP
ping -c 4 10.20.2.71

# From Server A: test TCP/SSH port reachability
 timeout 5 bash -c '</dev/tcp/10.20.2.71/22' && echo 'SSH PORT OPEN' || echo 'SSH PORT FAILED'

# From Server A: actual SSH login
ssh -i ~/.ssh/ewb-server-b-key -o IdentitiesOnly=yes ec2-user@10.20.2.71
```

From Server B, test Server A:

```bash
ping -c 4 10.10.1.147
```

A successful ping tests ICMP reachability, while a successful TCP check tests port reachability. Successful SSH login additionally requires correct key authentication.

## Troubleshooting

- **Ping fails in only one direction:** Check the destination instance's inbound ICMP security-group rule and subnet network ACLs. Do not assume the peering route is broken based on ping alone.
- **SSH port open, but `Permission denied (publickey...)`:** The network path works, but authentication failed. Confirm the right public key is in the target user's `~/.ssh/authorized_keys`, and check ownership and permissions.
- **SSM terminal user is `ssm-user`:** This is not necessarily the Linux login user used for SSH. When SSH is targeting `ec2-user`, place the public key in `/home/ec2-user/.ssh/authorized_keys`, not `/home/ssm-user/.ssh/authorized_keys`.
- **No SSM connection:** Verify the instance IAM role includes `AmazonSSMManagedInstanceCore`, SSM Agent is running, and the instance can reach the required Systems Manager endpoints (through NAT or VPC endpoints).
- **No route:** Ensure the peering connection is active and both route tables have the opposite VPC CIDR targeting the peering connection.

## Cleanup

To avoid ongoing AWS charges, terminate lab resources when no longer needed. Review and delete the EC2 instances, NAT Gateway, Elastic IP (if allocated), and other billable resources only after confirming you no longer need them. NAT Gateways can incur hourly and data-processing charges.

## Suggested GitHub repository description

`AWS VPC Peering lab with two VPCs, EC2 instances, private routing, SSM access, security groups, and verified SSH connectivity.`
