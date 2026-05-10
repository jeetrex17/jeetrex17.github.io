---
# the default layout is 'page'
icon: fas fa-laptop-code
order: 2
---

<style>
  .project-grid .card {
    background-color: var(--card-bg);
    color: var(--text-color);
    border: 1px solid var(--main-border-color, rgba(128, 128, 128, 0.2));
  }
  .project-grid .card-footer {
    background-color: transparent;
    border-top: 1px solid var(--main-border-color, rgba(128, 128, 128, 0.15));
  }
  .project-grid .card-title,
  .project-grid .card-text {
    color: var(--text-color);
  }
  .project-grid code {
    background-color: var(--mask-bg);
    color: var(--text-color);
    padding: 0.1em 0.35em;
    border-radius: 3px;
    font-size: 0.85em;
  }
  .project-grid .badge.bg-secondary {
    background-color: var(--mask-bg) !important;
    color: var(--text-muted-color, var(--text-color));
  }
</style>

Things I've built or am building. More to come as I learn.
{: .text-muted }

<div class="row row-cols-1 row-cols-md-2 g-4 mt-3 project-grid">

  <div class="col">
    <div class="card h-100 shadow-sm">
      <div class="card-body">
        <h5 class="card-title">🐚 Mini Shell in C</h5>
        <p class="card-text">A minimal Unix-like shell built from scratch in C — implements <code>fork</code>, <code>exec</code>, <code>wait</code>, and a few built-ins (<code>cd</code>, <code>help</code>, <code>exit</code>).</p>
        <p class="card-text"><small class="text-muted">C · systems · Linux</small></p>
      </div>
      <div class="card-footer bg-transparent border-0">
        <a href="https://github.com/jeetrex17/Mini_shell_in_c" class="btn btn-outline-primary btn-sm" target="_blank" rel="noopener">GitHub</a>
        <a href="/posts/making-shell-in-c/" class="btn btn-outline-secondary btn-sm">Write-up</a>
      </div>
    </div>
  </div>

  <div class="col">
    <div class="card h-100 shadow-sm">
      <div class="card-body">
        <h5 class="card-title">🧮 Order Book Simulator <small class="text-muted">— WIP</small></h5>
        <p class="card-text">A toy limit-order-book matching engine in C++. Goal: understand how exchanges match buys/sells under the hood, then measure latency.</p>
        <p class="card-text"><small class="text-muted">C++ · finance · low-latency</small></p>
      </div>
      <div class="card-footer bg-transparent border-0">
        <span class="badge bg-secondary">coming soon</span>
      </div>
    </div>
  </div>

  <div class="col">
    <div class="card h-100 shadow-sm">
      <div class="card-body">
        <h5 class="card-title">🧠 Memory Allocator from Scratch <small class="text-muted">— WIP</small></h5>
        <p class="card-text">A custom <code>malloc</code> / <code>free</code> implementation in C using <code>sbrk</code> and a free list. Notes coming after I finish reading <em>CSAPP</em>.</p>
        <p class="card-text"><small class="text-muted">C · systems · memory</small></p>
      </div>
      <div class="card-footer bg-transparent border-0">
        <span class="badge bg-secondary">coming soon</span>
      </div>
    </div>
  </div>

</div>
