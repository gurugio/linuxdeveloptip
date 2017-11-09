road to software-defined storage

1. read book
- Unix filesyatems, ch13 Clustered and Distributed Filesystems
- NFS illistrated

2. online course
- follow http://dirkmeister.blogspot.de/2010/01/storage-systems-course-my-own-idea.html
- select one course from http://dirkmeister.blogspot.de/2009/12/storage-system-and-file-system-courses.html

'''
Introduction, Overview, Disk Drive Architecture
Material: Ruemmler, Wilkes An introduction to disk drive modeling

Disk Scheduling / SSD
Material: Iyer, Druschel. Anticipatory scheduling: A disk scheduling framework to overcome deceptive idleness in synchronous I/O, Agrawal et al. Design Tradeoffs for SSD Performance

RAID
Material: Patterson et al. Introduction to Redundant Arrays of Inexpensive Disk (RAID), Corbett. Row-Diagonal Parity for Double Disk Failure Correction

Local File Systems

Local File System Case Studies: ext3, btrfs
Material: Valerie Aurora. A short history of btrfs, Card et al. Design and Implementation of the Second Extended Filesystem

Local File Structures (Sequential, Hashing, B-Tree)
Material: Comer. The Ubiquitous B-Tree

SAN / NAS / Object-based Storage
Material: Sacks. Demystifying DAS, SAN, NAS, NAS Gateways, Fibre Channel, and iSCSI

Examples: NFS, Ceph, GoogleFS/Hadoop DFS
Material: Weil. Ceph, A scalable, high-performance distributed file system, Ghemawat et al. The Google File System

Snapshots and Log-based Storage Designs
Material: Brinkmann, Effert. Snapshots and Continuous Data Replication in Cluster Storage Environments, Hitz et al. File System Design for an NFS File Server Appliance, Rosenblum, Ousterhout. The Design and Implementation of a Log-Structured File System

Fault Tolerance, Journaling, and Soft Updates
Material: Prabhakaran et al. Analysis and Evolution of Journaling File Systems, Seltzer et al. Journaling Versus Soft Updates: Asynchronous Meta-data Protection in File Systems

Advanced Hashing: Consistent Hashing, Share, and Crush
Material: Karger et al. Consistent hashing and random trees: distributed caching protocols for relieving hot spots on the World Wide Web, Weil et al. CRUSH: controlled, scalable, decentralized placement of replicated data

Caching, Replication
Material: Nelson et al. Caching in the Sprite network file system, Kistler et al. Disconnected operation in the Coda File System

Consistency, Availability, and Partition Tolerance
Material: DeCandia et al. Dynamo: Amazon’s Highly Available Key-value Store, Helland, Life beyond Distributed Transaction: An Apostate's Opinion

Data Deduplication
Material: Muthitacharoen et al., A Low-bandwidth Network File System, Douglis, Iyengar. Application-specific Delta-encoding via Resemblance Detection

Performance Analysis
Material: Traeger, A nine year study of file system and storage benchmarking (at least parts of it)
'''

Reading "Mastering CEPH"
