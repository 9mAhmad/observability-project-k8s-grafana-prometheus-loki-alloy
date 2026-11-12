# observability-project-ec2-k8s-grafana-prom-loki

## Work Log:  

### Preparing Ubuntu, Docker and Kubernetes in EC2

- Instead of Amazon Linux I have selected Ubuntu 26. I still have credits left, so using the largest free EC2 I can find because I will be installing many things on this:  
  <img width="1173" height="495" alt="image" src="https://github.com/user-attachments/assets/0df80b68-7975-42e9-bfc8-dff1239c5c96" />   

-  Taking a 30 GB volume instead of the default 8GB.
-  I could start the ubuntu machine and install everything 1 by 1. But I can also pass a script in the [user-data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html#userdata-linux) field. It can be shell script  or yaml also. [more docs](https://docs.cloud-init.io/en/24.1/reference/examples.html):
  <img width="689" height="188" alt="image" src="https://github.com/user-attachments/assets/e055b3ff-2965-46d0-bb42-14df3ee75987" />
  - refer to the ec2_user_data_field.yaml file.
- Verification of installations...
