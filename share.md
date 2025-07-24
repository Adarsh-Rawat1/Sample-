Internship Presentation: Dynamic Dashboard for DART


---

Slide 1: Title Slide

Title: Internship Project: Dynamic Dashboard for DART

Subtitle: Real-time Visualization of Trades, Usage, and Errors

Presented by: Adarsh Rawat | MCA | Graphic Era University



---

Slide 2: Project Overview

Built an internal dashboard using Blazor (.NET 8) to monitor:

📈 Trade activity

🧾 Stock usage patterns

⚠️ Error rates


Data sourced from Oracle Database

Replaced static analysis with real-time visual insights



---

Slide 3: Tech Stack & Evolution

Phase 1 (Initial Setup):

Python scripts + SQL → Matplotlib & Seaborn charts

Charts saved as PNGs → Served in frontend

Manual and limited scalability


Phase 2 (Dynamic System):

Blazor + PlotlyJS

Graph metadata + SQL defined in folder (via config)

Pages generated dynamically per graph

Simple to update: Just drop SQL + metadata




---

Slide 4: Optimizing Performance with Caching

Problem:

Querying DB at runtime slows down page (some take 10–20s+)


Solution:

Introduced caching system

Store results in local JSON cache file

On page load, render from this cache (instant)

Refresh logic:

🕒 Automatic (hourly/daily/weekly via polling)

🔁 Manual via "Refresh" button

♻️ Cooldown period defined in config





---

Slide 5: Creativity & Workarounds

DB does not allow notification/event triggers (due to policy)

Created a configurable polling mechanism instead

JSON defines: Graph type, query path, refresh interval


Switched from manual charting to fully dynamic component-driven system



---

Slide 6: Learnings & Takeaways

💻 Tech Skills: Python, SQL, C#, Blazor, PlotlyJS

⚙️ Concepts: Component systems, polling, caching, async rendering

🚧 Challenges: Query performance, chart rendering, system flexibility

✅ Outcome: Robust system that balances speed, accuracy, and usability



---

Slide 7: Thank You

"This project taught me how to turn raw backend data into meaningful, fast, and configurable dashboards."

Open to questions!




Your PPT content for the DART dashboard internship presentation is ready. It includes 7 focused slides aligned with the email instructions, your project work, and your experience.

Let me know if you want:

A PowerPoint file exported

Slide visuals like icons or diagrams added

A 5-minute speaking script for delivery


You’re good to go.

