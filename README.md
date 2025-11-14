# Byte-Stream-Reassembler
A robust C-based byte-stream reassembly engine. Implements a sorted linked list with merge-on-insert logic to handle variable-sized, out-of-order fragments with overlap detection.

This project was developed as an advanced data structures and algorithms project, focusing on dynamic memory management, complex linked list manipulation, and robust system design.

##  Core Features

* **Byte-Offset Reassembly:** Assembles packets using a byte-stream model.
* **Variable-Sized Fragment Handling:** Correctly handles fragments of any size, from a few bytes to thousands.
* **Sorted, Merging Linked List:** The core data structure is a sorted linked list of fragments. The system automatically performs a **"merge-on-insert"** to combine adjacent fragments.
* **Out-of-Order & Overlap Rejection:**
    * Handles out-of-order fragments seamlessly via sorted insertion.
    * Performs **robust overlap detection** to reject any fragment that attempts to overwrite existing data, ensuring data integrity.
* **Resource Limiting & Validation:**
    * Enforces a `MAX_PACKET_SIZE_BYTES` on registration.
    * Enforces a `MAX_PACKETS_IN_SYSTEM` limit to prevent resource exhaustion.
    * Validates all numeric input to prevent crashes.
* **Timeout Pruning:** A "garbage collector" (`system_prune_timeouts`) runs to find and free memory from packets that have been idle for too long.
* **Memory Safe:** All memory is manually and dynamically managed (`malloc`, `realloc`, `free`). The system is designed to be 100% free of memory leaks via its `free_assembler` and `system_cleanup` functions.

##  Data Structure Design

The system uses two levels of linked lists to manage its state. This separation of concerns is key to the design.



### 1. System-Level List

* **Struct:** `DefragmenterSystem`
* **Data:** `PacketNode* head`
* **Data Structure:** A standard **Singly Linked List**.
* **Purpose:** This list tracks all the `PacketAssembler`s that are currently "in-progress." A linked list was chosen for its **dynamic storage**, as we don't know if we'll be assembling 5 or 128 packets at once.
* **Trade-off:** This provides $O(1)$ insertion of new packets but requires an **$O(n)$ search** to find a packet by its ID.

### 2. Packet-Level List

* **Struct:** `PacketAssembler`
* **Data:** `FragmentNode* fragment_list_head`
* **Data Structure:** A **Sorted Singly Linked List**.
* **Purpose:** This is the core of the project. A simple array *fails* when fragments have variable sizes. This sorted list is the *only* structure that provides the flexibility to:
    1.  **Handle Variable Sizes:** A `FragmentNode` can store data of any length.
    2.  **Handle Out-of-Order:** The `assembler_add_fragment` function performs a **sorted insertion** based on the fragment's `offset`.
    3.  **Enable Merging:** A sorted list is essential to check if adjacent fragments "touch" and can be merged into a single, contiguous block of data.

##  Core Algorithm: `assembler_add_fragment()`

This is the most complex function and the heart of the engine. It performs a sorted, merging insert in $O(n)$ time.

1.  **Validate Bounds:** Checks if `offset + length > total_size_expected`. Rejects if true.
2.  **Find Position (O(n)):** Traverses the `fragment_list_head` to find the correct sorted insertion point, keeping `prev` and `curr` pointers.
3.  **Check Overlap (O(n)):** This is the critical data-integrity check. It rejects the fragment if it overlaps with `prev` or `curr` (e.g., `if (prev != NULL && (prev->offset + prev->length) > offset)`).
4.  **Check LFF:** Rejects if this fragment has the Last Fragment Flag but one has *already* been received for this packet.
5.  **Insert (O(1)):** Inserts the new `FragmentNode` into its sorted position in the list.
6.  **Merge Logic (O(n)):** After insertion, a `while` loop runs to check if the new node *touches* its neighbor.
    * If `(node->offset + node->length) == next_node->offset`, they are adjacent.
    * The function uses `realloc` to grow the first node's data buffer, `memcpy`s the second node's data onto the end, and `free`s the second node.
    * This logic is what allows the `assembler_is_complete` check to work.

### Completion Check

A packet is only considered "complete" when **all four** of these conditions are met:
1.  The `last_fragment_seen` flag is `true`.
2.  The `total_received_bytes` counter equals the `total_size_expected`.
3.  The fragment list has been fully merged into **one single node** (`fragment_list_head->next == NULL`).
4.  That single node's `offset` is `0`.

##  How to Compile & Run

This project consists of three files: `defrag.h`, `defrag.c`, and `main.c`.

### Compile
This code is C99 compliant. Compile using `gcc`:
```bash
gcc main.c defrag.c -o reassembler.exe
