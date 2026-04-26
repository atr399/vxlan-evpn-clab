# Operations Guide

Daily commands for running and managing this lab.

## The 4 Commands You Use Most

| What I want | Command |
|---|---|
| Start the lab | `sudo containerlab deploy -t vxlan-fabric.clab.yml` |
| Stop the lab | `sudo containerlab destroy -t vxlan-fabric.clab.yml --cleanup` |
| See lab status | `sudo containerlab inspect -t vxlan-fabric.clab.yml` |
| SSH to a device | `ssh admin@<ip>` (password: admin) |

## Daily Routine

### Start working

```bash
cd ~/labs/vxlan-fabric
sudo containerlab deploy -t vxlan-fabric.clab.yml
```

Wait 10-15 minutes. Check it's ready:

```bash
sudo containerlab inspect -t vxlan-fabric.clab.yml
```

### End of day (free RAM)

```bash
sudo containerlab destroy -t vxlan-fabric.clab.yml --cleanup
```

Your saved configs survive. Next day, just run deploy again.

## Saving Changes

When you change device config and want to keep it for next time:

```bash
cd ~/labs/vxlan-fabric

# Step 1: Tell containerlab to grab current config from devices
sudo containerlab save -t vxlan-fabric.clab.yml

# Step 2: Copy them to the safe folder
for node in spine1 spine2 leaf1 leaf2; do
  sudo cp clab-vxlan-fabric/$node/config/startup-config.cfg saved-configs/${node}.cfg
done
sudo chown -R $USER:$USER saved-configs/
```

Now if you destroy and redeploy, your changes come back.

## Backup to GitHub

Saving locally is good. Saving to GitHub is better - you cannot lose it.

After saving (above), also do:

```bash
git add saved-configs/
git commit -m "describe what you changed"
git push
```

Example commit messages:
- "Add VRF tenant-a on both leaves"
- "Increase MTU to 9000 on underlay"
- "Working baseline before BFD experiment"

## Going Back to a Working State

### Option A: Discard changes you have not saved yet

You edited something but realize it is wrong. Nothing committed yet.

```bash
cd ~/labs/vxlan-fabric
git checkout -- saved-configs/
```

Then redeploy to apply:

```bash
sudo containerlab destroy -t vxlan-fabric.clab.yml --cleanup
sudo containerlab deploy -t vxlan-fabric.clab.yml
```

### Option B: Roll back to a previous commit

You committed something that broke the lab. Want to go back to a known-working version.

```bash
git log --oneline
```

Output looks like:

```
a3d7e9f Broken VRF experiment
ffb214d Working VXLAN-EVPN fabric with 2 spines, 2 leaves
```

Roll back to the working one:

```bash
git reset --hard ffb214d
git push --force
```

(Use the actual hash from your `git log`.)

Then redeploy:

```bash
sudo containerlab destroy -t vxlan-fabric.clab.yml --cleanup
sudo containerlab deploy -t vxlan-fabric.clab.yml
```

### Option C: Pull latest from GitHub

You messed up local files badly. Just want to start over from what is on GitHub.

```bash
cd ~/labs/vxlan-fabric
sudo containerlab destroy -t vxlan-fabric.clab.yml --cleanup
git fetch origin
git reset --hard origin/main
sudo containerlab deploy -t vxlan-fabric.clab.yml
```

## Safe Experimentation Pattern

Before trying something risky:

```bash
# Save your current good state
cd ~/labs/vxlan-fabric
sudo containerlab save -t vxlan-fabric.clab.yml
for node in spine1 spine2 leaf1 leaf2; do
  sudo cp clab-vxlan-fabric/$node/config/startup-config.cfg saved-configs/${node}.cfg
done
sudo chown -R $USER:$USER saved-configs/

git add saved-configs/
git commit -m "Working baseline before experiment"
git push
```

Now experiment freely. If it works, save again. If it breaks, just roll back to this commit.

## Creating a Second Lab

Each lab needs its own folder.

```bash
mkdir -p ~/labs/my-second-lab
cd ~/labs/my-second-lab

# Copy the topology and configs as a starting point
cp -r ~/labs/vxlan-fabric/saved-configs ./saved-configs
cp ~/labs/vxlan-fabric/vxlan-fabric.clab.yml ./my-second-lab.clab.yml
```

Edit the file:

```bash
nano my-second-lab.clab.yml
```

Change the name on line 1 from `vxlan-fabric` to `my-second-lab`.

Deploy:

```bash
sudo containerlab deploy -t my-second-lab.clab.yml
```

Note: only one full fabric can run at a time on 32 GB RAM. Stop the first lab before starting the second.

## Common Problems

### Lab takes forever to boot

Normal. NX-OS needs 5-8 minutes per node to fully come up. Wait 15 minutes total.

### SSH says "host identification has changed"

Containerlab reuses IPs across deploys. Clear the old key:

```bash
ssh-keygen -f ~/.ssh/known_hosts -R <ip>
```

### Lab will not deploy: "startup-config not found"

You probably ran `--cleanup` which deleted `clab-vxlan-fabric/`. If the topology points there, it breaks.

Fix: make sure your `vxlan-fabric.clab.yml` uses `saved-configs/` (which survives cleanup), not `clab-vxlan-fabric/`.

### BGP not Established / NVE peer not up after deploy

Wait longer (full convergence takes 5+ minutes after boot completes). If still broken after 20 minutes, redeploy:

```bash
sudo containerlab destroy -t vxlan-fabric.clab.yml --cleanup
sudo containerlab deploy -t vxlan-fabric.clab.yml
```

## Glossary

- **Lab**: the whole topology (4 NX-OS + 2 hosts + links)
- **Deploy**: start the lab (boot all containers)
- **Destroy**: stop the lab (kill all containers)
- **--cleanup**: also delete the working directory (`clab-vxlan-fabric/`)
- **saved-configs/**: where your real configs live (survives cleanup)
- **commit**: snapshot your changes locally with git
- **push**: upload commits to GitHub
- **reset --hard**: throw away all local changes and go back to a saved state
