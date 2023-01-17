#!/bin/bash
# Written by Heiko Horn on 2019.08.06
# This script check all computer objects in Jamf to see if there are any failed MDM commands.
# Poorly updated on 01/16/2023 by Kyle
# Searches for Pending instead of Failed
# Only adds to list if count of pendings is greater than or equal to 8


# Script Variables
intCount=0
arrFailed=()



username=""
password=""
url="https://INSTANCE.jamfcloud.com"

#Variable declarations
bearerToken=""
tokenExpirationEpoch="0"

getBearerToken() {
	response=$(curl -s -u "$username":"$password" "$url"/api/v1/auth/token -X POST)
	bearerToken=$(echo "$response" | plutil -extract token raw -)
	tokenExpiration=$(echo "$response" | plutil -extract expires raw - | awk -F . '{print $1}')
	tokenExpirationEpoch=$(date -j -f "%Y-%m-%dT%T" "$tokenExpiration" +"%s")
}

checkTokenExpiration() {
    nowEpochUTC=$(date -j -f "%Y-%m-%dT%T" "$(date -u +"%Y-%m-%dT%T")" +"%s")
    if [[ tokenExpirationEpoch -gt nowEpochUTC ]]
    then
        echo "Token valid until the following epoch time: " "$tokenExpirationEpoch"
    else
        echo "No valid token available, getting new token"
        getBearerToken
    fi
}

invalidateToken() {
	responseCode=$(curl -w "%{http_code}" -H "Authorization: Bearer ${bearerToken}" $url/api/v1/auth/invalidate-token -X POST -s -o /dev/null)
	if [[ ${responseCode} == 204 ]]
	then
		echo "Token successfully invalidated"
		bearerToken=""
		tokenExpirationEpoch="0"
	elif [[ ${responseCode} == 401 ]]
	then
		echo "Token already invalid"
	else
		echo "An unknown error occurred invalidating the token"
	fi
}

checkTokenExpiration
curl -s -H "Authorization: Bearer ${bearerToken}" $url/api/v1/jamf-pro-version -X GET
checkTokenExpiration
#invalidateToken
#curl -s -H "Authorization: Bearer ${bearerToken}" $url/api/v1/jamf-pro-version -X GET

function clearFailedMdmCommands () {
	xmlresult=$(/usr/bin/curl -sk -H "Authorization: Bearer ${bearerToken}" -H 'Accept: application/xml' "${jamfUrl}/"JSSResource/commandflush/computers/id/"${id}"/status/Failed -X DELETE)
}

function getFailedMdmCommands () {
	xmlresult=$(/usr/bin/curl -sk -H "Authorization: Bearer ${bearerToken}" -H 'Accept: application/xml' "${jamfUrl}/"JSSResource/computerhistory/id/"${id}"/subset/Commands -X GET -H "accept: application/xml" | xmllint -xpath "/computer_history/commands/pending" - | /usr/bin/grep "</pending>")
}

function getAllComputers () {
	ids+=($(/usr/bin/curl -sk -H "Authorization: Bearer ${bearerToken}" -H 'Accept: application/xml' "${jamfUrl}/JSSResource/computers" -X GET -H "accept: application/xml" | xmllint --format - | awk -F'>|<' '/<id>/{print $3}' | sort -n))
}

function getComputerName () {
	name=$(/usr/bin/curl -sk -H "Authorization: Bearer ${bearerToken}" -H 'Accept: application/xml' "${jamfUrl}"/JSSResource/computers/id/"${id}" -X GET -H "accept: application/xml" | xmllint -xpath "/computer/general/name" - | sed -e 's/<[^>]*>//g')
}

function sendBlankPush () {
	xmlresult=$(/usr/bin/curl -sk -H "Authorization: Bearer ${bearerToken}" -H 'Accept: application/xml' "${jamfUrl}/"JSSResource/computercommands/command/BlankPush/id/"${id}" -X POST)
}

getAllComputers

#for id in "${ids[@]}"; do
#    echo "ID LIST: ${id}"
#done
#exit 1
testArray=(1816 1817 1818)

#for id in "${testArray[@]}"; do
for id in "${ids[@]}"; do
	echo "Getting failed commands for: ${id}"	
	getFailedMdmCommands

	# Clear failed MDM commands if they exist
	if [ ! -z "${xmlresult}" ]; then
		# Clear faild commands if they are VPP related
		#strFailed=$(echo $xmlresult | xmllint -xpath /failed/command/status - | /usr/bin/grep "VPP")
		#if [ ! -z "${strFailed}" ]; then
			#/bin/echo "Removing failed MDM commands ..."	
			#clearFailedMdmCommands
			#/bin/echo "Sending blank push notification ..."		
			#sendBlankPush
			#((intCount+=1))
			#getComputerName
			#arrFailed+=(${name})
			#echo -e "Computer name: ${name}\n"
		#fi
        echo "$xmlresult" > record.xml
        if [ `perl -nle "print s/<status>Pending//g" < record.xml | awk '{total += $1} END {print total}'` -ge 8 ]; then
            echo "GREATER"
            echo "XML:\n $xmlresult\n"
            ((intCount+=1))
            getComputerName
            pendingID+=(${id})
            arrFailed+=(${name})
            echo -e "Computer name: ${name}\n"
        fi
        #perl -nle "print s/<status>Pending//g" < record.xml | awk '{total += $1} END {print total}'
        #echo "Occurences: $ocurrences"
        #echo "XML:\n $xmlresult\n"
        #((intCount+=1))
        #getComputerName
        #arrFailed+=(${name})
        #echo -e "Computer name: ${name}\n"
	fi
done

echo -e "\nFound ${intCount} with failed commands\n"
if [[ ${intCount} > 0 ]]; then
	echo "Failed computers: ${arrFailed[@]}"
    printf '%s\n' "${arrFailed[@]}"
    printf '%s\n' "${pendingID[@]}"
fi

exit 0
