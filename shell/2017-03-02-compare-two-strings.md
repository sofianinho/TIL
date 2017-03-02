# How to compare two strings
## TL;DR
```sh
#list all content of /etc
var1=$(echo "$(find /etc  2>/dev/null)")
#list all content of /etc except file os-release
var2=$(echo "$(find /etc ! -name "os-release" 2>/dev/null)")
# compare those strings 
comm -23  <(echo $var1 |sed 's/ /\n/g'|sort) <(echo $var2 |sed 's/ /\n/g'|sort)
```
### Detailing the comparison part
This command:
```sh
echo $var1 |sed 's/ /\n/g'|sort
```
gives the content of the string `var1` and repalces the whitespaces by linebreaks and sorts alphabetically the output. 
This is due to the affectation part in command:
```sh
var1=$(echo "$(find /etc  2>/dev/null)")
```
where those linebreaks were lost (we need those to feed the `sort` command).

## The `comm` command itself
This program is part of the GNU coreutils (GNU core utilities). All useful information can be found locally using `info coreutils` or `man comm`
`comm` compares two sorted files, hence the sorting in the example before. Other similar and useful utilities are `join` and `uniq`.

## Another example
Check whether two folders have the same content and what has one that the other does not.
```sh
comm -23 <(echo "$(ls folder1)" |sort) <(echo "$(ls folder2)" |sort)
```
## Note
In these examples, we probably used `comm` in a different way from what it was intended originally. Other commands can be combined to "properly" compare strings.
