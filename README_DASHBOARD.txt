UNIVERSAL DASHBOARD
===================

Use these files at the top level:

  dashboard.html
    Main dashboard app. Open this in your browser.

  run_dashboard.command
    macOS launcher. Double-click it or run it from Terminal.

  run_dashboard.bat
    Windows launcher. Double-click it on Windows.

Folder layout:

  docs/
    Notes, build prompts, and architecture documentation.

  dist/
    Packaged/sharing copy of the dashboard.
    - dist/DASHBOARD.zip is the zip you can send to someone.
    - dist/DASHBOARD/ is the extracted copy contained in that zip.

Recommended use:

  For your own machine:
    Open dashboard.html from this main folder.

  For sharing:
    Send dist/DASHBOARD.zip.

Notes:

  - The dashboard runs as a plain HTML app.
  - Internet is needed for CDN libraries and fonts.
  - Claude API mode is optional. You can continue without an API key and use the local dashboard generator.
