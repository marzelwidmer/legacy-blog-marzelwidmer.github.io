# Notes

## Openshift
### Clean Up
CleanUp POD`s  
``` 
oc get pods --all-namespaces --no-headers | awk '$4 == "Error" \                                                                                                                                                  333ms  Mon Sep  2 11:22:33 2019
      || $4 == "Completed" \
      || $4 == "DeadlineExceeded" \
      || $4 == "ContainerCannotRun" \
      {system("bash -c '\''oc delete pod -n "$1" "$2" '\''")}'

```




### Adam Blog is a minimal clear theme for Jekyll
![Adam Blog - Imac](https://github.com/artemsheludko/adam-blog/blob/master/assets/img/adam-blog-imac.jpg?raw=true)
