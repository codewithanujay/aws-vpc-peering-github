# Verification Results

The lab was verified in the AWS Mumbai region (`ap-south-1`). These results reflect the terminal outputs observed during the session.

| Test | Result |
|---|---|
| Server B (`10.20.2.71`) ping to Server A (`10.10.1.147`) | Passed: 4 packets received, 0% loss after ICMP was allowed |
| Server A (`10.10.1.147`) ping to Server B (`10.20.2.71`) | Passed: 4 packets received, 0% loss after ICMP was allowed |
| Server A TCP connection to Server B port 22 | Passed: `SSH PORT OPEN` |
| SSH authentication from Server A to Server B | Passed: prompt changed to `ec2-user@ip-10-20-2-71` |
| SSM access to Server B | Passed |

Note: The initial ping and SSH authentication attempts failed before security-group rules and the correct `authorized_keys` location were fixed. These are useful troubleshooting examples, not final status.
