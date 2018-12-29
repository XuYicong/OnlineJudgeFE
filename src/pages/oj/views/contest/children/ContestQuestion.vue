<template>
  <div>
    <div id="ContestQuestionPanel">
      <Panel shadow>
        <div slot="title">Contest Question</div>
        <div slot="extra">
          <ul class="filter">
            <li>
              <Button type="primary" @click="get_question_list">Refresh</Button>
            </li>
          </ul>
        </div>
        <Table :data="DispQuestion" :columns="columns" :loading="loadingTable" disabled-hover></Table>
        <Pagination :total="total"
                    :pageSize="limit"
                    :current.sync="page"
                    @on-change="handlePage"></Pagination>
      </Panel>
    </div>
    <Panel shadow>
      <div slot="title">Create Question</div>
      <Form :model="CreateQuestionModel" :label-width="80" id="create-form">
        <FormItem label="Problem">
          <label>
            <Select v-model="create_question_problem">
              <Option v-for="item in problem_list" :value="item" :key="item">
                {{item}}
              </Option>
            </Select>
          </label>
        </FormItem>
        <FormItem label="Question">
          <Input v-model="create_question_text" type="textarea" :autosize="{minRows: 3,maxRows: 6}"
                 placeholder="Enter Question..."></Input>
        </FormItem>
        <FormItem>
          <Button type="primary" @click="submit_question">Submit</Button>
        </FormItem>
      </Form>
    </Panel>
  </div>
</template>

<script>
  import time from '@/utils/time'
  import {mapGetters, mapActions} from 'vuex'
  import {USER_TYPE} from '@/utils/constants'
  import Pagination from '@oj/components/Pagination.vue'
  import api from '@oj/api'

  export default {
    name: 'ContestQuestion',
    components: {
      Pagination
    },
    data () {
      return {
        limit: 5,
        page: 1,
        total: 0,
        loadingTable: false,
        create_question_problem: '',
        create_question_text: '',
        contestID: '',
        problem_list: [],
        DispQuestion: [],
        QuestionInfo: [],
        columns: [
          {
            title: 'Id',
            align: 'center',
            render: (h, params) => {
              return h('span', params.row.id)
            },
            width: 60
          },
          {
            title: 'When',
            align: 'center',
            render: (h, params) => {
              return h('span', time.utcToLocal(params.row.public_time))
            },
            width: 150
          },
          {
            title: 'Who',
            align: 'center',
            render: (h, params) => {
              return h('span', params.row.que_auth.username)
            },
            width: 100
          },
          {
            title: 'Problem',
            align: 'center',
            render: (h, params) => {
              return h('span', params.row.problem)
            },
            width: 200
          },
          {
            title: 'Question',
            align: 'center',
            render: (h, params) => {
              return h('span', {
                style: {
                  'max-height': '10px'
                }
              }, params.row.question)
            }
          },
          {
            title: 'Answer',
            align: 'center',
            render: (h, params) => {
              return h('span', params.row.answer)
            }
          }, {
            title: 'Status',
            width: 100,
            render: (h, params) => {
              if (params.row.is_solved) {
                return h('i-circle', {
                  props: {
                    'percent': 100,
                    'size': 35,
                    'stroke-width': 7,
                    'stroke-color': '#5cb85c'
                  }
                }, [
                  h('Icon', {
                    props: {
                      'type': 'ios-checkmark-empty',
                      'size': 40,
                      'color': '#5cb85c'
                    }
                  })
                ])
              } else {
                return h('i-circle', {
                  props: {
                    'percent': 100,
                    'size': 35,
                    'stroke-width': 7,
                    'stroke-color': '#268eb8'
                  }
                }, [
                  h('Icon', {
                    props: {
                      'type': 'ios-help-empty',
                      'size': 40
                    },
                    style: {
                      'color': '#268eb8'
                    }
                  })
                ])
              }
            }
          }
        ],
        CreateQuestionModel: {
          CreateQuestion_Problem: '',
          CreateQuestion_Question: ''
        },
        has_admin_column: false,
        answer_question_text: '',
        answer_question_is_public: false,
        answer_question_is_solved: false
      }
    },
    methods: {
      ...mapActions(['getContestProblems']),
      get_problem_list () {
        if (this.problem_list.length === 0) {
          this.getContestProblems().then(res => {
            this.problem_list.push('General')
            for (let v of res.data.data) {
              this.problem_list.push(v._id + ' - ' + v.title)
            }
          })
        }
      },
      submit_question () {
        if (this.create_question_problem.length === 0 || this.create_question_text.length === 0) {
          return this.$error('Please fill Empty Case.')
        }
        let param = {
          contest_id: this.contestID,
          question: this.create_question_text,
          problem: this.create_question_problem
        }
        api.ContestPostQuestion(param).then(() => {
          this.$success('Success')
          this.get_question_list()
        }, () => {
        })
      },
      showQuestionList () {
        this.loadingTable = true
        this.handlePage()
        this.loadingTable = false
      },
      handlePage (page = 1) {
        if (page !== 1) {
          this.loadingTable = true
        }
        let pageInfo = this.QuestionInfo.slice((this.page - 1) * this.limit, this.page * this.limit)
        for (let v of pageInfo) {
          if (v.init) {
          } else {
            v.init = true
          }
        }
        this.DispQuestion = pageInfo
        this.loadingTable = false
      },
      get_question_list () {
        this.contestID = this.$route.params.contestID
        api.ContestGetQuestionList(this.contestID).then(res => {
          let data = res.data.data
          let range = []
          for (let v of data) {
            range.push(v)
          }
          this.QuestionInfo = range
          this.total = range.length
          this.showQuestionList()
        }, () => {
        })
      },
      submit_answer (userquestion) {
        let data = {
          contest_id: this.contestID,
          qid: userquestion.id,
          answer: this.answer_question_text,
          is_solved: this.answer_question_is_solved,
          is_public: this.answer_question_is_public
        }
        api.ContestAnswerQuestion(data).then(() => {
          this.$success('Successful')
          this.get_question_list()
        }, () => {
        })
      },
      HandleAnswerForm (userquestion) {
        this.answer_question_is_public = userquestion.is_public
        this.answer_question_is_solved = userquestion.is_solved
        this.answer_question_text = userquestion.answer
        let _this = this
        this.$Modal.confirm({
          title: 'Answer Question',
          width: '600px',
          okText: 'Submit',
          onOk () {
            _this.submit_answer(userquestion)
          },
          render: (h) => {
            return [
              h('div', {
                style: {
                  'margin-top': '15px'
                }
              }, [
                h('span', {
                  style: {
                    'font-size': '15px',
                    'font-weight': 'bold'
                  }
                }, 'User: '),
                h('span', {
                  style: {
                    'font-size': '15px'
                  }
                }, userquestion.que_auth.username)
              ]),
              h('div', {
                style: {
                  'font-weight': 'bold',
                  'font-size': '15px'
                }
              }, 'Question: '),
              h('div', {
                style: {
                  'word-break': 'break-all'
                }
              }, userquestion.question),
              h('div', {
                style: {
                  'margin-top': '20px',
                  'word-break': 'break-all',
                  'font-size': '15px',
                  'font-weight': 'bold'
                }
              }, ('Last answer author: ' + ((userquestion.ans_auth === null) ? 'No Answer' : userquestion.ans_auth.username))),
              h('div', {
                style: {
                  'font-size': '15px',
                  'margin-top': '20px',
                  'font-weight': 'bold'
                }
              }, 'Your Answer'),
              h('i-input', {
                props: {
                  type: 'textarea',
                  row: 4,
                  value: _this.answer_question_text,
                  autofocus: true,
                  placeholder: 'Please enter your answer...'
                },
                on: {
                  input: (val) => {
                    _this.answer_question_text = val
                  }
                },
                style: {
                  'margin': '10px'
                }
              }),
              h('div', {
                style: {
                  'margin': '10px'
                }
              }, [
                h('span', {
                  style: {
                    'font-size': '15px'
                  }
                }, 'Public: '),
                h('i-switch', {
                  props: {
                    value: _this.answer_question_is_public
                  },
                  on: {
                    'on-change': (val) => {
                      _this.answer_question_is_public = val
                    }
                  }
                })
              ]),
              h('div', {
                style: {
                  'margin': '10px'
                }
              }, [
                h('span', {
                  style: {
                    'font-size': '15px'
                  }
                }, 'Solved: '),
                h('i-switch', {
                  props: {
                    value: _this.answer_question_is_solved
                  },
                  on: {
                    'on-change': (val) => {
                      _this.answer_question_is_solved = val
                    }
                  }
                })
              ])
            ]
          }
        })
      },
      add_admin_column () {
        if (this.has_admin_column || !this.ContestQuestionColumnVisible) {
          return
        }
        const AdminColumn = {
          title: 'Admin Option',
          fixed: 'right',
          align: 'center',
          width: 150,
          render: (h, params) => {
            return h('div', [
              h('Button', {
                props: {
                  type: 'primary',
                  size: 'small'
                },
                style: {
                  marginRight: '5px'
                },
                on: {
                  click: () => {
                    this.HandleAnswerForm(params.row)
                    this.get_question_list()
                  }
                }
              }, 'Answer'),
              h('Button', {
                props: {
                  type: 'error',
                  size: 'small'
                },
                on: {
                  click: () => {
                    api.ContestDeleteQuestion(this.contestID, params.row.id).then(() => {
                      this.$success('Successful')
                      this.get_question_list()
                    }, () => {
                    })
                  }
                }
              }, 'Delete')
            ])
          }
        }
        this.columns.push(AdminColumn)
        this.has_admin_column = true
      }
    },
    mounted () {
      this.get_problem_list()
      this.add_admin_column()
      this.get_question_list()
    },
    computed: {
      ...mapGetters(
        ['user']
      ),
      ContestQuestionColumnVisible () {
        return this.$route.params.contestID && this.user.admin_type === USER_TYPE.SUPER_ADMIN
      }
    }
  }
</script>

<style scoped>

  /*
    暂时高度应用于子元素 防止问题过长
  */
  #ContestQuestionPanel >>> .ivu-table-cell {
    max-height: 150px !important;
  }

  #ContestQuestionPanel {
    margin-bottom: 70px;
  }

  #create-form {
    padding: 30px;
  }
</style>
