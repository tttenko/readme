```java

{
  "expand": "schema,names",
  "startAt": 0,
  "maxResults": 10,
  "total": 10,
  "issues": [
    {
      "expand": "operations,versionedRepresentations,editmeta,changelog,renderedFields",
      "id": "99999902",
      "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issue/99999902",
      "key": "CROSSGOAL-999902",
      "fields": {
        "summary": "FR2: обновление существующей инициативы и monitoring — проверка 05.10",
        "lastViewed": null,
        "created": "2026-10-03T10:00:00.000+0300",
        "customfield_31306": "5",
        "customfield_31305": null,
        "description": "Тестовый сценарий 02-update: обновление полей, связей, monitoring Epic, Task, QG и SLA.",
        "reporter": {
          "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/user?username=19995362",
          "name": "19995362",
          "key": "JIRAUSER1219644",
          "emailAddress": "NIBubentsova@sberbank.ru",
          "avatarUrls": {
            "48x48": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?avatarId=21415",
            "24x24": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=small&avatarId=21415",
            "16x16": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=xsmall&avatarId=21415",
            "32x32": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=medium&avatarId=21415"
          },
          "displayName": "Бубенцова Наталья Игоревна",
          "active": true,
          "timeZone": "Etc/GMT-3"
        },
        "customfield_31307": "5",
        "customfield_31304": "21,5 FTE",
        "customfield_30401": "456,78",
        "labels": [
          "AI_Native_портфель"
        ],
        "customfield_30001": [
          "КИБ/Corporate_Lending"
        ],
        "customfield_34300": "345,67",
        "customfield_30002": [
          "Оптимизация_ПШЕ_шт."
        ],
        "customfield_15903": [
          {
            "name": "GigaChat Max",
            "checked": true,
            "mandatory": false,
            "option": false,
            "id": -1,
            "rank": 0,
            "status": null
          }
        ],
        "customfield_29202": null,
        "resolutiondate": null,
        "customfield_29205": "П4905",
        "issuelinks": [
          {
            "id": "32761537",
            "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issueLink/32761537",
            "type": {
              "id": "10300",
              "name": "PartOf",
              "inward": "consists of",
              "outward": "is part of",
              "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issueLinkType/10300"
            },
            "outwardIssue": {
              "id": "19246646",
              "key": "CROSSGOAL-55639",
              "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issue/19246646",
              "fields": {
                "summary": "Стратегическая тема: Появление ИИ-помощника",
                "status": {
                  "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/status/18805",
                  "description": "",
                  "iconUrl": "https://atlassianstack.sigma.sbrf.ru/jirasilver/images/icons/statuses/generic.png",
                  "name": "Анализ ценности",
                  "id": "18805",
                  "statusCategory": {
                    "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/statuscategory/4",
                    "id": 4,
                    "key": "indeterminate",
                    "colorName": "inprogress",
                    "name": "In Progress"
                  }
                },
                "priority": {
                  "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/priority/4",
                  "iconUrl": "https://atlassianstack.sigma.sbrf.ru/jirasilver/images/icons/priorities/minor.svg",
                  "name": "Minor",
                  "id": "4"
                },
                "issuetype": {
                  "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issuetype/10749",
                  "id": "10749",
                  "description": "",
                  "iconUrl": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/viewavatar?size=xsmall&avatarId=21401&avatarType=issuetype",
                  "name": "Инициатива",
                  "subtask": false,
                  "avatarId": 21401
                }
              }
            }
          },
          {
            "id": "32761538",
            "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issueLink/32761538",
            "type": {
              "id": "10300",
              "name": "PartOf",
              "inward": "consists of",
              "outward": "is part of",
              "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issueLinkType/10300"
            },
            "inwardIssue": {
              "id": "19375246",
              "key": "GIGAUSAGE-7631",
              "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issue/19375246",
              "fields": {
                "summary": "Автоматическая проверка отлагательного условия по договору при рассмотрении заявки на выдачу кредитных средств",
                "status": {
                  "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/status/3",
                  "description": "",
                  "iconUrl": "https://atlassianstack.sigma.sbrf.ru/jirasilver/images/icons/statuses/generic.png",
                  "name": "In Progress",
                  "id": "3",
                  "statusCategory": {
                    "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/statuscategory/4",
                    "id": 4,
                    "key": "indeterminate",
                    "colorName": "inprogress",
                    "name": "In Progress"
                  }
                },
                "priority": {
                  "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/priority/4",
                  "iconUrl": "https://atlassianstack.sigma.sbrf.ru/jirasilver/images/icons/priorities/minor.svg",
                  "name": "Minor",
                  "id": "4"
                },
                "issuetype": {
                  "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issuetype/10749",
                  "id": "10749",
                  "description": "",
                  "iconUrl": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/viewavatar?size=xsmall&avatarId=21401&avatarType=issuetype",
                  "name": "Инициатива",
                  "subtask": false,
                  "avatarId": 21401
                }
              }
            }
          },
          {
            "id": "99999904",
            "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issueLink/99999904",
            "type": {
              "id": "10300",
              "name": "PartOf",
              "inward": "consists of",
              "outward": "is part of",
              "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issueLinkType/10300"
            },
            "inwardIssue": {
              "id": "99999903",
              "key": "CROSSGOAL-999903",
              "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issue/99999903",
              "fields": {
                "summary": "Мониторинг портфеля AI-Native",
                "status": {
                  "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/status/10109",
                  "description": "",
                  "iconUrl": "https://atlassianstack.sigma.sbrf.ru/jirasilver/images/icons/subtask.gif",
                  "name": "To Do",
                  "id": "10109",
                  "statusCategory": {
                    "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/statuscategory/2",
                    "id": 2,
                    "key": "new",
                    "colorName": "default",
                    "name": "To Do"
                  }
                },
                "priority": {
                  "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/priority/4",
                  "iconUrl": "https://atlassianstack.sigma.sbrf.ru/jirasilver/images/icons/priorities/minor.svg",
                  "name": "Minor",
                  "id": "4"
                },
                "issuetype": {
                  "id": "10000",
                  "name": "Epic",
                  "subtask": false
                }
              }
            }
          }
        ],
        "customfield_29203": null,
        "assignee": null,
        "updated": "2026-10-05T23:59:00.000+0300",
        "status": {
          "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/status/10506",
          "description": "Это статус для нового артефакта в JIRA. В таком статусе поступает запрос на доработку",
          "iconUrl": "https://atlassianstack.sigma.sbrf.ru/jirasilver/images/icons/statuses/generic.png",
          "name": "Новый",
          "id": "10506",
          "statusCategory": {
            "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/statuscategory/2",
            "id": 2,
            "key": "new",
            "colorName": "default",
            "name": "To Do"
          }
        },
        "resolution": null,
        "issuetype": {
          "id": "10749",
          "name": "Инициатива",
          "subtask": false
        }
      }
    },
    {
      "id": "99998910",
      "key": "CROSSGOAL-998910",
      "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issue/99998910",
      "fields": {
        "summary": "Этап: Концепция",
        "description": "Тестовая monitoring Task для CROSSGOAL-999902.",
        "issuetype": {
          "id": "3",
          "name": "Task",
          "subtask": false
        },
        "status": {
          "id": "10110",
          "name": "Done",
          "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/status/10110",
          "statusCategory": {
            "id": 3,
            "key": "done",
            "name": "Done"
          }
        },
        "created": "2026-10-03T10:00:00.000+0300",
        "updated": "2026-10-05T23:59:00.000+0300",
        "lastViewed": null,
        "customfield_16700": "2026-10-03T10:00:00.000+0300",
        "customfield_16701": "2026-10-06T18:00:00.000+0300",
        "resolutiondate": "2026-10-05T11:00:00.000+0300",
        "resolution": {
          "id": "1",
          "name": "Fixed"
        },
        "customfield_10008": "CROSSGOAL-999903",
        "labels": [],
        "assignee": null,
        "reporter": {
          "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/user?username=19995362",
          "name": "19995362",
          "key": "JIRAUSER1219644",
          "emailAddress": "NIBubentsova@sberbank.ru",
          "avatarUrls": {
            "48x48": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?avatarId=21415",
            "24x24": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=small&avatarId=21415",
            "16x16": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=xsmall&avatarId=21415",
            "32x32": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=medium&avatarId=21415"
          },
          "displayName": "Бубенцова Наталья Игоревна",
          "active": true,
          "timeZone": "Etc/GMT-3"
        }
      }
    },
    {
      "id": "99998911",
      "key": "CROSSGOAL-998911",
      "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issue/99998911",
      "fields": {
        "summary": "Этап: PoC",
        "description": "Тестовая monitoring Task для CROSSGOAL-999902.",
        "issuetype": {
          "id": "3",
          "name": "Task",
          "subtask": false
        },
        "status": {
          "id": "3",
          "name": "In Progress",
          "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/status/3",
          "statusCategory": {
            "id": 4,
            "key": "indeterminate",
            "name": "In Progress"
          }
        },
        "created": "2026-10-03T10:00:00.000+0300",
        "updated": "2026-10-05T23:59:00.000+0300",
        "lastViewed": null,
        "customfield_16700": "2026-10-03T10:00:00.000+0300",
        "customfield_16701": "2026-10-07T18:00:00.000+0300",
        "resolutiondate": null,
        "resolution": null,
        "customfield_10008": "CROSSGOAL-999903",
        "labels": [],
        "assignee": null,
        "reporter": {
          "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/user?username=19995362",
          "name": "19995362",
          "key": "JIRAUSER1219644",
          "emailAddress": "NIBubentsova@sberbank.ru",
          "avatarUrls": {
            "48x48": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?avatarId=21415",
            "24x24": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=small&avatarId=21415",
            "16x16": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=xsmall&avatarId=21415",
            "32x32": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=medium&avatarId=21415"
          },
          "displayName": "Бубенцова Наталья Игоревна",
          "active": true,
          "timeZone": "Etc/GMT-3"
        }
      }
    },
    {
      "id": "99998912",
      "key": "CROSSGOAL-998912",
      "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issue/99998912",
      "fields": {
        "summary": "Этап: MVP",
        "description": "Тестовая monitoring Task для CROSSGOAL-999902.",
        "issuetype": {
          "id": "3",
          "name": "Task",
          "subtask": false
        },
        "status": {
          "id": "10109",
          "name": "To Do",
          "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/status/10109",
          "statusCategory": {
            "id": 2,
            "key": "new",
            "name": "To Do"
          }
        },
        "created": "2026-10-03T10:00:00.000+0300",
        "updated": "2026-10-05T23:59:00.000+0300",
        "lastViewed": null,
        "customfield_16700": "2026-10-03T10:00:00.000+0300",
        "customfield_16701": "2026-10-08T18:00:00.000+0300",
        "resolutiondate": null,
        "resolution": null,
        "customfield_10008": "CROSSGOAL-999903",
        "labels": [],
        "assignee": null,
        "reporter": {
          "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/user?username=19995362",
          "name": "19995362",
          "key": "JIRAUSER1219644",
          "emailAddress": "NIBubentsova@sberbank.ru",
          "avatarUrls": {
            "48x48": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?avatarId=21415",
            "24x24": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=small&avatarId=21415",
            "16x16": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=xsmall&avatarId=21415",
            "32x32": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=medium&avatarId=21415"
          },
          "displayName": "Бубенцова Наталья Игоревна",
          "active": true,
          "timeZone": "Etc/GMT-3"
        }
      }
    },
    {
      "id": "99998913",
      "key": "CROSSGOAL-998913",
      "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issue/99998913",
      "fields": {
        "summary": "Этап: Целевое решение в ПРОМ",
        "description": "Тестовая monitoring Task для CROSSGOAL-999902.",
        "issuetype": {
          "id": "3",
          "name": "Task",
          "subtask": false
        },
        "status": {
          "id": "10109",
          "name": "To Do",
          "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/status/10109",
          "statusCategory": {
            "id": 2,
            "key": "new",
            "name": "To Do"
          }
        },
        "created": "2026-10-03T10:00:00.000+0300",
        "updated": "2026-10-05T23:59:00.000+0300",
        "lastViewed": null,
        "customfield_16700": "2026-10-03T10:00:00.000+0300",
        "customfield_16701": "2026-10-09T18:00:00.000+0300",
        "resolutiondate": null,
        "resolution": null,
        "customfield_10008": "CROSSGOAL-999903",
        "labels": [],
        "assignee": null,
        "reporter": {
          "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/user?username=19995362",
          "name": "19995362",
          "key": "JIRAUSER1219644",
          "emailAddress": "NIBubentsova@sberbank.ru",
          "avatarUrls": {
            "48x48": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?avatarId=21415",
            "24x24": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=small&avatarId=21415",
            "16x16": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=xsmall&avatarId=21415",
            "32x32": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=medium&avatarId=21415"
          },
          "displayName": "Бубенцова Наталья Игоревна",
          "active": true,
          "timeZone": "Etc/GMT-3"
        }
      }
    },
    {
      "id": "99998914",
      "key": "CROSSGOAL-998914",
      "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issue/99998914",
      "fields": {
        "summary": "Этап: Обратная связь",
        "description": "Тестовая monitoring Task для CROSSGOAL-999902.",
        "issuetype": {
          "id": "3",
          "name": "Task",
          "subtask": false
        },
        "status": {
          "id": "10109",
          "name": "To Do",
          "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/status/10109",
          "statusCategory": {
            "id": 2,
            "key": "new",
            "name": "To Do"
          }
        },
        "created": "2026-10-03T10:00:00.000+0300",
        "updated": "2026-10-05T23:59:00.000+0300",
        "lastViewed": null,
        "customfield_16700": "2026-10-03T10:00:00.000+0300",
        "customfield_16701": "2026-10-10T18:00:00.000+0300",
        "resolutiondate": null,
        "resolution": null,
        "customfield_10008": "CROSSGOAL-999903",
        "labels": [],
        "assignee": null,
        "reporter": {
          "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/user?username=19995362",
          "name": "19995362",
          "key": "JIRAUSER1219644",
          "emailAddress": "NIBubentsova@sberbank.ru",
          "avatarUrls": {
            "48x48": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?avatarId=21415",
            "24x24": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=small&avatarId=21415",
            "16x16": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=xsmall&avatarId=21415",
            "32x32": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=medium&avatarId=21415"
          },
          "displayName": "Бубенцова Наталья Игоревна",
          "active": true,
          "timeZone": "Etc/GMT-3"
        }
      }
    },
    {
      "id": "99998915",
      "key": "CROSSGOAL-998915",
      "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issue/99998915",
      "fields": {
        "summary": "Концепция: Сформировано описание кейса: бизнес-логика, область внедрения (процесс / продукт), соответствие страт. целям",
        "description": "Тестовая monitoring Task для CROSSGOAL-999902.",
        "issuetype": {
          "id": "3",
          "name": "Task",
          "subtask": false
        },
        "status": {
          "id": "10110",
          "name": "Done",
          "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/status/10110",
          "statusCategory": {
            "id": 3,
            "key": "done",
            "name": "Done"
          }
        },
        "created": "2026-10-03T10:00:00.000+0300",
        "updated": "2026-10-05T23:59:00.000+0300",
        "lastViewed": null,
        "customfield_16700": "2026-10-03T10:00:00.000+0300",
        "customfield_16701": "2026-10-11T18:00:00.000+0300",
        "resolutiondate": "2026-10-05T11:00:00.000+0300",
        "resolution": {
          "id": "1",
          "name": "Fixed"
        },
        "customfield_10008": "CROSSGOAL-999903",
        "labels": [],
        "assignee": null,
        "reporter": {
          "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/user?username=19995362",
          "name": "19995362",
          "key": "JIRAUSER1219644",
          "emailAddress": "NIBubentsova@sberbank.ru",
          "avatarUrls": {
            "48x48": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?avatarId=21415",
            "24x24": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=small&avatarId=21415",
            "16x16": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=xsmall&avatarId=21415",
            "32x32": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=medium&avatarId=21415"
          },
          "displayName": "Бубенцова Наталья Игоревна",
          "active": true,
          "timeZone": "Etc/GMT-3"
        }
      }
    },
    {
      "id": "99998916",
      "key": "CROSSGOAL-998916",
      "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issue/99998916",
      "fields": {
        "summary": "Концепция: Сформирован план реализации",
        "description": "Тестовая monitoring Task для CROSSGOAL-999902.",
        "issuetype": {
          "id": "3",
          "name": "Task",
          "subtask": false
        },
        "status": {
          "id": "10110",
          "name": "Done",
          "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/status/10110",
          "statusCategory": {
            "id": 3,
            "key": "done",
            "name": "Done"
          }
        },
        "created": "2026-10-03T10:00:00.000+0300",
        "updated": "2026-10-05T23:59:00.000+0300",
        "lastViewed": null,
        "customfield_16700": "2026-10-03T10:00:00.000+0300",
        "customfield_16701": "2026-10-12T18:00:00.000+0300",
        "resolutiondate": "2026-10-05T11:00:00.000+0300",
        "resolution": {
          "id": "1",
          "name": "Fixed"
        },
        "customfield_10008": "CROSSGOAL-999903",
        "labels": [],
        "assignee": null,
        "reporter": {
          "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/user?username=19995362",
          "name": "19995362",
          "key": "JIRAUSER1219644",
          "emailAddress": "NIBubentsova@sberbank.ru",
          "avatarUrls": {
            "48x48": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?avatarId=21415",
            "24x24": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=small&avatarId=21415",
            "16x16": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=xsmall&avatarId=21415",
            "32x32": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=medium&avatarId=21415"
          },
          "displayName": "Бубенцова Наталья Игоревна",
          "active": true,
          "timeZone": "Etc/GMT-3"
        }
      }
    },
    {
      "id": "99998917",
      "key": "CROSSGOAL-998917",
      "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issue/99998917",
      "fields": {
        "summary": "PoC: Произведен baseline-расчет метрик, используемых для мониторинга",
        "description": "Тестовая monitoring Task для CROSSGOAL-999902.",
        "issuetype": {
          "id": "3",
          "name": "Task",
          "subtask": false
        },
        "status": {
          "id": "10110",
          "name": "Done",
          "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/status/10110",
          "statusCategory": {
            "id": 3,
            "key": "done",
            "name": "Done"
          }
        },
        "created": "2026-10-03T10:00:00.000+0300",
        "updated": "2026-10-05T23:59:00.000+0300",
        "lastViewed": null,
        "customfield_16700": "2026-10-03T10:00:00.000+0300",
        "customfield_16701": "2026-10-13T18:00:00.000+0300",
        "resolutiondate": "2026-10-05T11:00:00.000+0300",
        "resolution": {
          "id": "1",
          "name": "Fixed"
        },
        "customfield_10008": "CROSSGOAL-999903",
        "labels": [],
        "assignee": null,
        "reporter": {
          "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/user?username=19995362",
          "name": "19995362",
          "key": "JIRAUSER1219644",
          "emailAddress": "NIBubentsova@sberbank.ru",
          "avatarUrls": {
            "48x48": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?avatarId=21415",
            "24x24": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=small&avatarId=21415",
            "16x16": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=xsmall&avatarId=21415",
            "32x32": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=medium&avatarId=21415"
          },
          "displayName": "Бубенцова Наталья Игоревна",
          "active": true,
          "timeZone": "Etc/GMT-3"
        }
      }
    },
    {
      "id": "99998918",
      "key": "CROSSGOAL-998918",
      "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/issue/99998918",
      "fields": {
        "summary": "MVP: Утверждена концептуальная архитектура",
        "description": "Тестовая monitoring Task для CROSSGOAL-999902.",
        "issuetype": {
          "id": "3",
          "name": "Task",
          "subtask": false
        },
        "status": {
          "id": "3",
          "name": "In Progress",
          "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/status/3",
          "statusCategory": {
            "id": 4,
            "key": "indeterminate",
            "name": "In Progress"
          }
        },
        "created": "2026-10-03T10:00:00.000+0300",
        "updated": "2026-10-05T23:59:00.000+0300",
        "lastViewed": null,
        "customfield_16700": "2026-10-03T10:00:00.000+0300",
        "customfield_16701": "2026-10-14T18:00:00.000+0300",
        "resolutiondate": null,
        "resolution": null,
        "customfield_10008": "CROSSGOAL-999903",
        "labels": [],
        "assignee": null,
        "reporter": {
          "self": "https://atlassianstack.sigma.sbrf.ru/jirasilver/rest/api/2/user?username=19995362",
          "name": "19995362",
          "key": "JIRAUSER1219644",
          "emailAddress": "NIBubentsova@sberbank.ru",
          "avatarUrls": {
            "48x48": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?avatarId=21415",
            "24x24": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=small&avatarId=21415",
            "16x16": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=xsmall&avatarId=21415",
            "32x32": "https://atlassianstack.sigma.sbrf.ru/jirasilver/secure/useravatar?size=medium&avatarId=21415"
          },
          "displayName": "Бубенцова Наталья Игоревна",
          "active": true,
          "timeZone": "Etc/GMT-3"
        }
      }
    }
  ]
}


```
