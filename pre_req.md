## INSTALL THIS TO OBTAIN MEMORY INFO

'''sudo apt install dmidecode -y '''

## RUN THE BELOW COMMAND IN TERMINAL ONCE TO OBTAIN MEM CLOCKS
sudo dmidecode -t memory | awk -F: '/Configured Memory Speed/ {print $2;exit}; ' > ./RAM\_INFO
