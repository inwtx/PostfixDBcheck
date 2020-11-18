# Postfix DB check
Check that postfix DB files have been postmap updated.
  
The following bash script will check to see if a corresponding .DB file date and time are within 3 minutes of each other. It will send an email to an address of choice to notify that a required 'postmap' to update the .DB was not perormed.  
  
Cronjobs required:  
*/2 * * * * /etc/specialpgms/PostfixDbChecker.sh sender_access &> /dev/null  
*/2 * * * * /etc/specialpgms/PostfixDbChecker.sh recipient_access &> /dev/null  
  
```
#!/bin/bash
#
# PostfixDbCheck.sh
#
# This script checks to see if the postfix files that need to be postmap have
# been serviced (example: recipient_access/recipient_access.db).  If the
# date/time on a postfix files that need this is more than 2 minutes apart,
# this script will send an email warning to postmap the file.
# cron examples:
# */2 * * * * /path/to/PostfixDbCheck.sh sender_access &> /dev/null
# */2 * * * * /path/to/PostfixDbCheck.sh recipient_access &> /dev/null
# */2 * * * * /path/to/PostfixDbCheck.sh transport &> /dev/null
#

filePath=${0%/*}  # current file path

myemail="<your email address here>"


if [[ $myemail == "" ]] || [[ ! $myemail =~ "@" ]]; then
   echo ""
   echo " No email address found in 'myemail=' parameter within script."
   echo " Open script and put an email address in 'myemail=' parameter."
   echo ""
   exit 1
fi

echo "$1"

if [[ $1 == "" ]]; then
   echo ""
   echo " No postfix file following PostfixDbCheck.sh execute line."
   echo " Example: /path/to/PostfixDbCheck.sh recipient_access"
   echo ""
   exit 1
fi


cd /etc/postfix
echo "$(ls -lA)" > $filePath/PostfixDbChecker$1.txt
cd $filePath


grep "$1" $filePath/PostfixDbChecker$1.txt > $filePath/PostfixDbChecker$1.txt2
sed -i '/\_regexp/d' $filePath/PostfixDbChecker$1.txt2  # Get rid of "remove any regexp filename"

varRecPDC=""
varDbRecPDC=""

varRecPDC=$(grep -v ".db" $filePath/PostfixDbChecker$1.txt2)    # extract for line not containing .db '-rw-r--r-- 1 root root   496 Feb  9 07:19 sender_access'
varDbRecPDC=$(grep ".db" $filePath/PostfixDbChecker$1.txt2)     # extract for line containing .db     '-rw-r--r-- 1 root root 12288 Feb  9 07:22 sender_access.db'


if [[ ${#varRecPDC} -eq 0 ]] || [[ ${#varDbRecPDC} -eq 0 ]]; then  # if len = 0
   exit 0
fi

varYear=$(date -u +"%Y")  # get current year

sed -e '/^$/d' <<<$varRecPDC   # remove blank lines
sed -e '/^$/d' <<<$varDbRecPDC # remove blank lines#

varMonPDC=${varRecPDC:29:3}  #  extract month from rec - -rw-r--r-- 1 root root   496 'Feb'  9 07:19 sender_access
varDayPDC=${varRecPDC:33:2}  #  extract day from rec   - -rw-r--r-- 1 root root   496 Feb ' 9' 07:19 sender_access
varTimePDC=${varRecPDC:36:5} #  extract time from rec  - -rw-r--r-- 1 root root   496 Feb  9 '07:19' sender_access

varDbMonPDC=${varDbRecPDC:29:3} #  extract month from rec - -rw-r--r-- 1 root root   496 'Feb'  9 07:19 sender_access
varDbDayPDC=${varDbRecPDC:33:2}  #  extract day from rec   - -rw-r--r-- 1 root root   496 Feb ' 9' 07:19 sender_access
varDbTimePDC=${varDbRecPDC:36:5} #  extract time from rec  - -rw-r--r-- 1 root root   496 Feb  9 '07:19' sender_access

if [[ $(grep -c ":" <<<$varTimePDC) -eq 0 ]] || [[ $(grep -c ":" <<<$varDbTimePDC) -eq 0 ]]; then  # year in time slot = over year old
   rm $filePath/PostfixDbChecker$1.txt
   rm $filePath/PostfixDbChecker$1.txt1
   rm $filePath/PostfixDbChecker$1.txt2
   exit 0
fi

CURRENT=$(date +%s -d "$varDayPDC $varMonPDC $varYear $varTimePDC:00")
TARGET=$(date +%s -d "$varDbDayPDC $varDbMonPDC $varYear $varDbTimePDC:00")
MINUTES=$(( ( $CURRENT - $TARGET ) ))

if [[ $(printf '%d%d\n' $(($MINUTES/3600)) $(($MINUTES%3600/60))) -gt 3 ]]; then  # send error if recipient_access.db not updated within 5 minutes
   mail -s "From PostfixDbCheck.sh" $myemail <<<"$1 and $1.db files sync times are > 3 minutes. $varTimePDC:00 $varDbTimePDC:00"
#   mutt -s "From PostfixDbCheck.sh" $myemail <<<"$1 and $1.db files sync times are > 3 minutes. $varTimePDC:00 $varDbTimePDC:00"
fi

rm $filePath/PostfixDbChecker$1.txt
rm $filePath/PostfixDbChecker$1.txt1
rm $filePath/PostfixDbChecker$1.txt2

exit 0

```
