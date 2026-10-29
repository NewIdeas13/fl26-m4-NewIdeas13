# M4 Design Note

Due Wed Oct 28, 11:59 pm. See specification Section 7.

Components and threads

I plan to keep MainWindow, the Qt widgets, and the active Workspace and ProcessingCore on the GUI thread. When Build Corpus is clicked, MainWindow::build_corpus() will make a copy of the current workspace and start a background task using std::async(std::launch::async, ...). The worker will create its own ProcessingCore, apply the requested delay in small intervals, and rebuild the copied workspace. It will never access Qt widgets. A QTimer on the GUI thread will check whether the std::future is ready and handle the completed result or an exception. This avoids blocking the event loop.

The workspace copy and the worker's processing core belong only to the worker during the build. The GUI alone accesses the future. The only mutable object shared across threads will be a cancellation flag (std::atomic<bool>): the GUI writes it and the worker reads it, which is safe because the operations are atomic.

Corpus states and transitions

I will use four main states: Not Built, Building, Current, and Stale/Failed. Build Corpus changes the state to Building. A successful result becomes Current only if its saved workspace version still matches the present version. Adding a document marks the corpus stale and increases that version. Reset returns it to Not Built. A failed build is reported to the user and leaves processing disabled until another successful build. I will disable another Build Corpus request while one is running, but keep the rest of the GUI usable.

Risks and event sequences

The main risks are freezing the interface (R1), accepting old results (R2), losing errors (R3), and leaving work running during shutdown (R4).

Example 1: adding a document during a build: The worker starts with a copy of documents A and B and records version 1. The user adds document C, so MainWindow increments the version to 2, invalidates the corpus, and requests cancellation. If the old task still finishes, the timer sees that version 1 does not match version 2 and discards its result. A later build can include A, B, and C.

Example 2: Reset during a build: Reset clears the workspace and GUI outputs, increments the version, and requests cancellation. Even if the worker finishes afterward, its result is ignored. For Exit, the application also requests cancellation and waits for the owned background task to finish before destroying its state. No threads are detached.

When a build starts, the background worker creates and owns a temporary ProcessingCore containing the new corpus. Once finished, ownership is transferred to MainWindow through a future. If the corpus is still valid, it replaces the old one; otherwise, it is discarded. In either case, std::unique_ptr ensures the objects are automatically destroyed when no longer needed

The worker owns its temporary ProcessingCore while rebuilding. Once finished, ownership of that result moves through the future to MainWindow, which means if it is still valid, it replaces the active core. Otherwise the temporary result is destroyed. I rejected running rebuild() directly on the GUI thread because a long build would freeze the window and prevent users from interacting with it.