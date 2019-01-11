# SHELL IN DOCKER #

# 服务程序启动 #

->1.
	EX:

	RUN_COMMAMD=${BUILT_FILE}" >> "${LOG_FILE}" 2>&1 &"

	Start(){
		eval ${RUN_COMMAMD}
	}

	End(){
        SPID=`ps -ef | grep ${BUILT_FILE} | grep -v grep | awk '{print $2}'`
        kill ${SPID}
    }

    Start
	trap End 2 15

	while true
	do
	:
	done
