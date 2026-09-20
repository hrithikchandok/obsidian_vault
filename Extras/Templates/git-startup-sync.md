<%*
// Startup: wait for plugins to load, then trigger obsidian-git commit-and-sync (push).
try {
  await new Promise(r => setTimeout(r, 5000));
  const ok = app.commands.executeCommandById("obsidian-git:commit-push");
  if (!ok) {
    new Notice("Git startup sync: command not found (plugin disabled?)");
  } else {
    new Notice("Git startup sync triggered");
  }
} catch (e) {
  new Notice("Git startup sync error: " + e.message);
}
%>
