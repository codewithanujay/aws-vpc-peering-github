# AWS Deployment Checklist

- [ ] Create VPC A with CIDR `10.10.0.0/16`.
- [ ] Create VPC A public subnet `10.10.1.0/24` and configure its internet route if required.
- [ ] Launch Server A and note its private IP.
- [ ] Create VPC B with CIDR `10.20.0.0/16`.
- [ ] Create VPC B private subnet `10.20.2.0/24`.
- [ ] Launch Server B without a public IP.
- [ ] Attach an IAM role with `AmazonSSMManagedInstanceCore` to Server B.
- [ ] Create and accept VPC Peering.
- [ ] Add route `10.20.0.0/16` to the peering connection in VPC A's route table.
- [ ] Add route `10.10.0.0/16` to the peering connection in VPC B's private route table.
- [ ] Configure narrow security-group rules for the required traffic.
- [ ] Connect to Server B using Session Manager.
- [ ] Verify ping from B to A and A to B (if ICMP is allowed).
- [ ] Verify TCP port 22 from A to B.
- [ ] Configure SSH public-key authentication if required.
- [ ] Record verification results and remove temporary test rules.
- [ ] Review AWS resources and costs; clean up resources when finished.
