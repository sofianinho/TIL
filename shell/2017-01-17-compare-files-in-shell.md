#Compare files in a shell
```shell
_ret_code=$(cmp --silent file1 file2)
if [ "$_ret_code" -eq "0" ]; then
 echo "file1 and file2 are identitical"
else
 echo "file1 and file2 are different"
fi
```
More options of the `cmp` can be found in the manuel.

#Previously seen in 
I first saw this command in the docker entrypoint for the `gitlab/gitlab-runner:v1.1.4` image. It was used to update a certificate file (*ca.crt*) given as an argument. 

Basically, the initial image (should) already have something in the `/usr/local/share/ca-certificates/ca.crt` file. If the user when starting the image defines an environnement variable `CA_CERTIFICATES_PATH` (structured in certain way) that points to a user defined certificate, it is compared to the original in `/usr/local/share` then updated if different.

```sh
if [ -f "${CA_CERTIFICATES_PATH}" ]; then
  # update the ca if the custom ca is different than the current
  cmp --silent "${CA_CERTIFICATES_PATH}" "${LOCAL_CA_PATH}" || update_ca
fi
```
The `||` in this command would work as follows:
 - If return code of the first part is `0` (also know as `true` for boolean comparisons), that is to say that both files are identitical, then the second part is *not* executed (CA not updated).
 - If the return code for `cmp` is `1` however (AKA `false` in boolean), the second part of the boolean operation is "evaluated", i.e. the `update_ca` function executed. Which is only logical.

The original code inspired by this entry is [here](https://gitlab.com/gitlab-org/gitlab-ci-multi-runner/raw/master/dockerfiles/ubuntu/entrypoint) (written on January, 17th 2017). 



