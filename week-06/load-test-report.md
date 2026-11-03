# Week 6: Load Test Report

I put PropertyLite behind an Application Load Balancer with an Auto Scaling group of t3.micro instances in the private subnets of my VPC (desired 2, min 1, max 4). The group uses a target tracking policy that keeps average CPU at 50%. Through the load balancer, `/health` returned `{"status":"ok"}` and `/properties` returned the listings.

My first stress run (about 09:43 UTC) did not scale anything because I had not yet created the scaling policy, so no CloudWatch alarm existed. After the run ended, CPU dropped to near 0% and the policy's `AlarmLow` alarm shrank the group from 2 to 1 instance at 02:17:13 AM PST. I then re-ran the test, SSHing into both app servers through the bastion and running `stress-ng --cpu 2` on each.

- Stress started: 2:19:41 AM PST (10:19:41 UTC)
- CPU alarm (`AlarmHigh`) went into ALARM: 2:23:16 AM PST (10:23:16 UTC)
- Scale-out started (1 to 2 instances): 2:23:19 AM PST (10:23:19 UTC)
- Time gap from stress start to scale-out: 3 minutes 38 seconds

The CloudWatch graph (`cpu-spike.png`) shows CPU rising from near 0% to about <Y>%, and the Activity tab (`scale-out-activity.png`) shows a new instance launched by the AlarmHigh alarm. The gap comes from the alarm needing several consecutive one-minute datapoints above the 50% target before it fires, plus a short metric delay. Once the alarm fired, the launch began within about 3 seconds. Detailed monitoring (1-minute metrics) kept the delay short.