# A curated (limited) list of shell tools (programs) that I recently used
(no preferred order)

## tr
- `from man` 
  - translate or delete characters
- `How to use it` 
  - if you have a command that has multiple tabulations or spaces, replace them with only one in order to sed/awk/cut your way through the output to a desired information.
### Example
```sh
#get all the PIDs in your system
ps ax|tr -s [:space:]|sed '/^ /d'|cut -d' ' -f1
```
- `ps ax` get all the processes in your system
- `tr -s [:space:]` delete the repeating spaces and replace them with only one
- `sed '/^ /d'` delete spaces at the beginning of the line
- `cut -d' ' -f1` select only the first column (in this case, the PID column)
