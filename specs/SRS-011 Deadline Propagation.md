# SRS-011 Deadline Propagation

Using Timeouts in Cascading Calls (Deadline Propagation)

In the case of cascading calls, using consistent but static timeout values in each of the services in the chain is not always the most optimal solution. As an alternative solution, dynamic timeout values can be used through the deadline propagation technique. In this case, the time remaining to complete each operation in the cascade is passed along the chain, starting from the first service.

<img width="897" alt="Screenshot 2024-12-11 at 16 40 36" src="https://github.com/user-attachments/assets/ee94a5a2-5354-42a0-83fb-b6904b9d2977" />

---
* https://sre.google/sre-book/addressing-cascading-failures/

# TODO

* Agree on header names for timeouts and their management
