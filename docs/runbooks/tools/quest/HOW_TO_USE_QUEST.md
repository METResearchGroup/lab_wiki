Quest is a computing cluster, shared across all of Northwestern.

I'd suggest poking around the Quest website to learn a bit more about how it works, what you can/can't run on there (e.g., you won't be able to open any ports), etc. A good place to get started is https://rcdsdocs.it.northwestern.edu/start/quest/quest-intro.html

You'll probably want to SSH into Quest, at which point check out https://rcdsdocs.it.northwestern.edu/systems/quest/user-guide/login/login-quest.html#connect-with-ssh

Once you get logged on, check your working directory. Should be something like /home/{netid} iirc. You'll want to go to /projects/p32375/ and then git clone the repo, something like git clone https://github.com/METResearchGroup/lab_data_integrations_interface.git

To use the full power of Quest, you'll want to be able to submit jobs to Slurm (https://rcdsdocs.it.northwestern.edu/systems/quest/user-guide/slurm/slurm.html). Slurm is Quest's job scheduler - when you log into Quest, your actual user account has access to only a very small allocation (maybe 1GB?) and the full power of Quest comes from using its compute nodes to do stuff.
This is similar to other large-scale applications like Hadoop and Spark and such where the actual compute nodes that "do stuff" are managed by a scheduler, and it's up to you to define your code in a job that can be submitted to the scheduler, which gathers the compute nodes necessary for your task. I'd recommend experimenting with how this works and submitting a few jobs before actually kicking them off.
An example of one of these submission scripts (in addition to the resource linked above) is https://github.com/METResearchGroup/bluesky-research/blob/0399c54eba87787de88e67cbf0619e48b9fcd960/pipelines/classify_records/perspective_api/submit_job.sh#L12
A pattern that I suggest is running a long-running cron-job on your node that submits a 24 hour cron job to Quest, as it's easier to get a node allocation when you request nodes for less time (this is better than running the cron job on your personal node as Quest reserves the right to kill jobs at any time or reclaim compute if it's done in personal nodes rather than the compute nodes).
Try to not request compute nodes for too long or for too much memory. The scheduler works off a "fairness" system, so if you have a pattern of requesting too much memory or for too long, it kicks you to the back of the priority queue.

For the settings for the submission script, something like this should suffice:

#!/bin/bash

#SBATCH -A p32375
#SBATCH -p short
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=2
#SBATCH -t 2:00:00
#SBATCH --mem=10G
#SBATCH --mail-type=FAIL
#SBATCH --mail-user={your email}
#SBATCH --job-name={job id}_%j
#SBATCH --output=/projects/p32375/lab_data_integrations_interface/{logs path}/{log name}-%j.log

Some notes:
Use a VPN when accessing Quest. iirc I don't think it'll even let you use it if you don't have one.
Please be very careful with what you run. Any rm -rf will likely irreversibly delete data. For example, I never run Claude Code or any coding agents within Quest, as the risk of losing data (both your own and that of other researchers) is too high. I once saw a researcher rm -rf a lab's data and rack up a $50,000 bill as a result.
I'd suggest treating Quest as a blank computing cluster that just does stuff. What that means is that you should do any dev work locally, get it to work, and then have the exact commands that you need to run so you can just run it directly in Quest. You'll soon find out that writing code within Quest, or connecting Quest to your IDE via SSH, is quite a painful experience (especially if you have to use a VPN)
You won't have sudo access on Quest. I think there's a workaround where if you wanted to, you could do it, but there should be no reason to need escalated permissions on Quest.

