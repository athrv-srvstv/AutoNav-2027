To Build : 
"docker build -t ouster_robot .
"

To Run :
"docker run -it --rm --network host -u root -v ~/ouster_ws:/ouster_ws -w /ouster_ws ouster_robot
"
