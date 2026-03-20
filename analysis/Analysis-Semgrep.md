

## How to Run

```bash
python3 -m venv ~/security-venv
source ~/security-venv/bin/activate
pip install --upgrade pip
pip install semgrep
semgrep --version

semgrep --config "p/security-audit" --config "p/c" src/imagemagick-6.9.12.98+dfsg1/magick --json > semgrep_output.json

```



## Results

``` bash
┌─────────────┐
│ Scan Status │
└─────────────┘
  Scanning 6 files tracked by git with 276 Code rules:
                                                                                                                        
  Language      Rules   Files          Origin      Rules                                                                
 ─────────────────────────────        ───────────────────                                                               
  c                60       6          Community     225                                                                
  <multilang>       2       6          Pro rules      51                                                                
  cpp              51       4                                                                                           
                                                                                                                        
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 0:00:06                                                                                                                        
                
                
┌──────────────┐
│ Scan Summary │
└──────────────┘
✅ Scan completed successfully.
 • Findings: 3 (3 blocking)
 • Rules run: 62
 • Targets scanned: 6
 • Parsed lines: ~99.3%
 • Scan was limited to files tracked by git
 • For a detailed list of skipped files and lines, run semgrep with the --verbose flag
Ran 62 rules on 6 files: 3 findings.
```
