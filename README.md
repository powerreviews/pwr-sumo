pwr-sumo
=========

Sumologic on Amazon Linux/Redhat

Role Variables
--------------

Just need to run this role passing these variables as group_vars or playbook vars.

    sumologic_collector_accessid: ""
    sumologic_collector_accesskey: ""
    sumologic_collector_log_paths:
      - name: "logname"
        path: "log_path"
        use_multiline: true
        category: "log_category"
      - name: "logname2"
        path: "log_path2"
        use_multiline: true
        category: "log_category"

Example Playbook
----------------

    - name: Install SumoLogic Collector
      hosts: all
      roles:
        - pwr-sumo

License
-------

BSD

Author Information
------------------

Mario Harvey -- PowerReviews
