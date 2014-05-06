StrongBox
============
For execution help,
```bash
python StrongBox.py -h
```

## Demo Screencast
All in due time...

### What?

A working prototype of an encrypted, P2P file syncing system in the vein of OwnCloud (a self-hosted, client-server  system styled after the more famous DropBox). With StrongBox, users benefit from the accessibility and reliability of having redundant backups of their data without sacrificing privacy. Made as a final project submission for Professor Jim Waldo's Spring 2014 CS262 at Harvard (not bad for two weeks of coding!).

### Why?

With any "free" service, be it e-mail or file storage, there's always a hidden cost to make things worthwhile for the service provider. Generally, this payoff comes in the form of valuable information gleaned from analyzing users' data and potentially directed advertising based on that information. In other models, payed "premium" services are offered with paying users offsetting the cost of their own services and those of free-tier users.

StrongBox instead uses the peer-to-peer model so users can decide on their own terms for backing up one another's data. As originally conceived, this agreement would mean that each user offers the other some amount of storage space as well as the bandwidth required for synchronization and the electricity/maintenance costs associated with keeping their machine on. By syncing their data to other StrongBox peers, a user benefits from the additional accessibility and failure independence of redundant, geographically distributed backups.

Privacy is also of primary importance in StrongBox. In the wake of Edward Snowden's revelations about digital spying programs such as PRISM, and tech companies' complicity therein, users' concerns about online privacy continue to increase as their confidence in businesses' handling of private data continues to decline [[1]](http://www.truste.com/about-TRUSTe/press-room/news_us_truste_reveals_consumers_more_concerned_about_data_collection) [[2]](http://www.pewinternet.org/2013/09/05/anonymity-privacy-and-security-online/). It has also been well established that the inability to store and communicate private data has a "chilling effect" on democracy and free speech [[3]](https://www.eff.org/press/releases/eff-files-22-firsthand-accounts-how-nsa-surveillance-chilled-right-association) [[4]](http://www.presstv.com/detail/2013/11/12/334416/us-writers-scared-silent-by-nsa-spying/).

StongBox makes abundant use of strong cryptography to ensure the privacy and integrity of a user's synced data. All communications between peers occur over encrypted SSL channels to prevent eavesdropping or the manipulation of communications. More importantly, StrongBox seamlessly AES-256 encrypts a user's synced data (including file and directory names) before it ever leaves their machine, so no one without the user's private encryption key can discover the original contents. Furthermore, these guarantees don't come at the cost of convenience as is frequently the case when securing one's data. A user is free to modify their files as normal while StrongBox takes care of the synchronization in the background.


### How?

There are three primary concepts at work in the exectution of StrongBox: **stores**, **revisions**, and **peers**.

##### Stores

A store is the directory of files and subdirectories a user wants to have synced and will be backed up in its entirety to associated backup peers. On the store owner's machines, the store directory is just like any other and can have its contents read or modified at will. However, a peer backing up a store will receive it in encrypted form as all store data (including file names and directory names) is AES-256 encrypted before ever leaving its owner's machines. 

To verify that an encrypted backup hasn't been partially deleted or otherwise tampered with, backup integrity is checked using a SHA-256 Merkel tree (or hash tree) implementation for use with filesytem directories. When requesting verification from a backup peer, the store owner (alternatively another backup peer) will generate a random nonce (or "salt") with which the backup peer is to compute the overall hash value of the Merkel tree. If the backup peer's resulting hash matches the nonced hash computed by the requester, the integrity of the backup is cryptographically ensured. Merkel trees are also used to identify when two copies of a store are out of sync and the specific (encrypted) files and directories that differ between the two.


##### Revisions

A revision of a store can be thought of as a cohesive state of the store at some point in time. A store owner's StrongBox instance will monitor their store directory and upon detecting a change will signal for new revision data to be generated. Revisions are given numbers in the spirit of Lamport's logical clocks to give sequentiality to the changes a store undergoes. To these revision numbers are attached the overall Merkel tree hash for the store and an RSA-4096 digital signature covering both revision number and hash. The revision hash allows backup peers and the owner's other StrongBox instances to independently verify the integrity of their copy of the store. The digital signature prevents network errors or malicious peers from manipulating revision data. A malicious peer could still retransmit old signed data (a replay attack), but the presence of the revision number allows up to date peers to detect the stale data.

##### Peers

A peer is a running instance of StrongBox, syncing its owner's store. StrongBox peers interact directly with one another. During communication, peers "gossip" to one another about the state of other peers to quickly disseminate information across the system. For example, peer A might gossip to peer B, "peer C doesn't have a valid revision of store X and needs an update," or "peer D just entered the system network address Z and might be of interest to you." Through the course of communications, two peers will come to an agreement on which mutually held store they will sync on and what type of sync each will be undertaking (send, receive, or verify).


### What's missing?
* A chain of trust or certificate signing authority and cryptographically sound peer verification checks. Currently a peer can be imitated, however the imitator would only gain access to encrypted data and would be unable to convince other peers to modify the state of any previously known store.
* Automated backup association. As originally proposed, peers would bid storage space available to others and a central server would match peers with compatible bids and instruct the peers to act as backups for one-another. Currently, this is done on a uni-directional basis by moving a config file to the backing up peer via some side-band (e.g. copying via scp, sneakernet).
* Paxos. Currently, if a user updates their store on one of their machines and then makes subsequent changes on another of their machines that did not retrieve the original update (a "split brain" scenario), StrongBox will not be able to choose between the two and only the changes on the machine that makes the next (third) update will be kept. This could be remedied by requiring each revision to enact a Paxos-style two-phase commit e.g. using Doozer as a Paxos implementation.
* Staging sync changes before committing. This would allow a peer to verify the changes they received before moving them into place.