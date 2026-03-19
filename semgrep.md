```bash
python3 -m venv ~/security-venv
source ~/security-venv/bin/activate
pip install --upgrade pip
pip install semgrep
semgrep --version

(security-venv) alonso@alonso-Inspiron-7391:~/projects/Seguridad_del_Software-Proyecto1$ semgrep --config "p/security-audit" --config "p/c" src/imagemagick-6.9.12.98+dfsg1/magick --json > semgrep_output.json
               
               
┌─────────────┐
│ Scan Status │
└─────────────┘
  Scanning 256 files tracked by git with 276 Code rules:
                                                                                                                        
  Language      Rules   Files          Origin      Rules                                                                
 ─────────────────────────────        ───────────────────                                                               
  <multilang>       2     256          Community     225                                                                
  c                60     245          Pro rules      51                                                                
  cpp              51     149                                                                                           
                                                                                                                        
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 0:01:41                                                                                                                        
                
                
┌──────────────┐
│ Scan Summary │
└──────────────┘
✅ Scan completed successfully.
 • Findings: 6 (6 blocking)
 • Rules run: 62
 • Targets scanned: 256
 • Parsed lines: ~99.5%
 • Scan was limited to files tracked by git
 • For a detailed list of skipped files and lines, run semgrep with the --verbose flag
Ran 62 rules on 256 files: 6 findings.

```
