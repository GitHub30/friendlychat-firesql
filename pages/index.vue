<template>
  <div class="container">
    <div>
      <p v-for="message in messages" :key="message.id">
        {{ message.message }} ({{ message.username }})
      </p>
    </div>
    <form @submit.prevent="insert">
      <!-- eslint-disable-next-line vue/html-self-closing -->
      <input v-model="newMessage" type="text" />
      <button>送信</button>
    </form>
  </div>
</template>

<script>
import io from 'socket.io-client'

class FireSQL {
  constructor(url) {
    // eslint-disable-next-line no-console
    console.log('initialize')
    this.socket = io(url)
  }

  on(table, callback) {
    this.socket.emit('_enter_room', table)
    this.socket.on(table, callback)
  }

  query(sql) {
    // eslint-disable-next-line no-console
    console.log(sql)
    // eslint-disable-next-line prettier/prettier
    const promise = new Promise(resolve => this.socket.emit('_query', sql, resolve))
    this._clearSQL()
    return promise
  }

  _clearSQL() {
    this.tableName = ''
    this.whereString = ''
  }

  table(tableName) {
    this.tableName = tableName
    return this
  }

  where(...args) {
    if (args.length === 2) args = [args[0], '=', args[1]]
    const condition = `${args[0]}${args[1]}${this._escape(args[2])}`
    if (this.whereString) {
      this.whereString += ` AND ${condition}`
    } else {
      this.whereString = ` WHERE ${condition}`
    }
    return this
  }

  select(...columns) {
    const columnsString = columns.length ? columns.join(', ') : '*'
    const sql = `SELECT ${columnsString} FROM ${this.tableName}`
    // eslint-disable-next-line prettier/prettier
    return new Promise(resolve => this.query(sql).then(resolve))
  }

  insert(...valuesArray) {
    const columnsString = Object.keys(valuesArray[0]).join(', ')
    const valuesArrayString = valuesArray
      // eslint-disable-next-line prettier/prettier
      .map(values => `(${Object.values(values).map(this._escape).join(', ')})`)
      .join(', ')
    const sql = `INSERT INTO ${this.tableName} (${columnsString}) VALUES ${valuesArrayString}`
    // eslint-disable-next-line prettier/prettier
    return new Promise(resolve => this.query(sql).then(resolve))
  }

  update(values) {
    const valuesString = Object.entries(values)
      .map(([column, value]) => `${column}=${this._escape(value)}`)
      .join(', ')
    const sql = `UPDATE ${this.tableName} SET ${valuesString}${this.whereString}`
    // eslint-disable-next-line prettier/prettier
    return new Promise(resolve => this.query(sql).then(resolve))
  }

  delete() {
    const sql = `DELETE FROM ${this.tableName}${this.whereString}`
    // eslint-disable-next-line prettier/prettier
    return new Promise(resolve => this.query(sql).then(resolve))
  }

  _escape(data) {
    switch (typeof data) {
      case 'string':
        return `'${data.replace(/'/g, "''")}'`
      default:
        return data
    }
  }
}

export default {
  components: {},
  data() {
    return {
      messages: [],
      newMessage: '',
      firesql: null
    }
  },
  mounted() {
    this.firesql = new FireSQL('http://localhost:8080')
    window.firesql = this.firesql
    this.firesql.on('messages', this.onMessages)
    // eslint-disable-next-line no-console
    this.firesql.on('messages/173', (row, type) => console.log(row, type))
    // eslint-disable-next-line prettier/prettier
    this.firesql.table('messages').select().then((messages) => { this.messages = messages })
    // 'CREATE TABLE IF NOT EXISTS messages (message_id int NOT NULL AUTO_INCREMENT, username VARCHAR(255), message VARCHAR(255), PRIMARY KEY(message_id))'
    // "UPDATE messages SET message = 'レドックスウォーター' WHERE (message_id = '181');"
    // 'INSERT INTO messages (username, message) VALUES ("山田太郎😊", "こんばんわに")'
  },
  methods: {
    onMessages(rows, type) {
      // eslint-disable-next-line no-console
      console.log(rows, type)
      if (type === 'WriteRowsEvent') {
        // eslint-disable-next-line prettier/prettier
        this.messages.push(...rows.map(row => row.values))
      } else if (type === 'UpdateRowsEvent') {
        for (const row of rows) {
          const index = this.getIndexByMessageID(row.before_values.message_id)
          if (index !== -1) this.messages.splice(index, 1, row.after_values)
        }
      } else if (type === 'DeleteRowsEvent') {
        for (const row of rows) {
          const index = this.getIndexByMessageID(row.values.message_id)
          if (index !== -1) this.messages.splice(index, 1)
        }
      }
    },
    getIndexByMessageID(messageId) {
      // eslint-disable-next-line prettier/prettier
      return this.messages.map(message => message.message_id).indexOf(messageId)
    },
    insert() {
      // eslint-disable-next-line no-console
      console.log('insert')
      const username = navigator.userAgent.includes('Chrome')
        ? '山田太郎😊'
        : '日本花子💞'
      this.firesql
        .table('messages')
        .insert({ username, message: this.newMessage })
    }
  }
}
</script>

<style>
.container {
  margin: 0 auto;
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  text-align: center;
}

.title {
  font-family: 'Quicksand', 'Source Sans Pro', -apple-system, BlinkMacSystemFont,
    'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  display: block;
  font-weight: 300;
  font-size: 100px;
  color: #35495e;
  letter-spacing: 1px;
}

.subtitle {
  font-weight: 300;
  font-size: 42px;
  color: #526488;
  word-spacing: 5px;
  padding-bottom: 15px;
}

.links {
  padding-top: 15px;
}
</style>
