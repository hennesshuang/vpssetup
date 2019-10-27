#!/usr/bin/bash

function install_vnstat {
    vnstat=$(which vnstat)
    if [ $? -eq 0 ]; then
        echo "vnstat already installed."
        return
    fi
    
    apt-get install vnstat
    if [ $? -eq 0 ]; then
        echo "Successfully installed vnstat"
    else
        echo "Could not install vnstat"
        exit 1
    fi

    service vnstat start
}

function select_network_inetrface {
    # Show Network Interface
    ip address
    default_interface=$(ip -br address | grep -v ^lo | cut -d' ' -f 1 | head -n 1)
    echo -n "Please select a network interface to monitory ($default_interface): "
    read network_interface
    if [[ ! $network_interface ]]; then
        network_interface=$default_interface
    fi
}

function set_traffic_limit {
    echo -n "Please input network traffic total monthly (999) GiB: "
    read traffic_limit
    re='^[0-9]+$'
    if ! [[ $traffic_limit =~ $re ]] ; then
        traffic_limit=999
    fi
}

function set_check_period {
    echo -n "Please input network traffic check period (3) min: "
    read check_period
    re='^[0-9]+$'
    if ! [[ $check_period =~ $re ]] ; then
        check_period=3
    fi
}


function create_check_trafic {
(
cat <<EOF
#!/usr/bin/bash

traffic_limit=$traffic_limit
current_traffic=\$(vnstat -i $network_interface --oneline | cut -d ';' -f 11)
current_date=\$(date +"%Y-%m-%d %R")
echo "\$current_date: \$current_traffic" >> /var/log/network_traffic.log

if [[ "\$current_traffic" == *GiB* ]]; then
    current_traffic_number=\$(echo \$current_traffic | cut -d ' ' -f 1)
    if (( \$(echo "\$current_traffic_number > \$traffic_limit" | bc -l) )); then
        shutdown -h now
    fi
fi
EOF
) > check_traffic.sh
}

function edit_crontab {
    crontab_file=current_crontab.txt
    # crontab_file=/etc/crontab
    crontab -u root -l > current_crontab.txt
    sed -i '/check_traffic/d' $crontab_file
    echo "*/$check_period * * * * /usr/bin/check_traffic.sh" >> $crontab_file
    crontab -u root $crontab_file
    service cron reload
}

echo Step 1. Install vnstat is a console-based network traffic monitor
install_vnstat
echo

echo Step 2. Select Network Interface
select_network_inetrface
vnstat -u -i $network_interface
current_traffic=$(vnstat -i $network_interface --oneline | cut -d ';' -f 11)
echo network traffic total for current month: $current_traffic
echo

echo Step 3. Set Network Monthly Limit
set_traffic_limit
echo

echo Step 4. Set Network Traffic Check Period
set_check_period
echo

echo Step 5. Create check_traffic.sh
create_check_trafic
echo

echo Step 6. Add check.traffic.sh to chrontab
cp check_traffic.sh /usr/bin/
chmod +x /usr/bin/check_traffic.sh
edit_crontab
echo

echo Step 7. Delete temporary file
rm current_crontab.txt
rm check_traffic.sh
echo

echo "LOG: /var/log/network_traffic.log"
