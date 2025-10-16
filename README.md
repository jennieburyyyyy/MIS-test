# MIS-test
import streamlit as st
import sqlite3
import pandas as pd

st.set_page_config(page_title="个人信息管理系统", layout="wide")
st.title("👤 个人信息管理系统")
st.markdown("---")

# 初始化数据库
conn = sqlite3.connect('personal_info.db', check_same_thread=False)
cursor = conn.cursor()

# 创建两个表
cursor.execute('''
CREATE TABLE IF NOT EXISTS persons(
                   id INTEGER PRIMARY KEY AUTOINCREMENT,
                   name TEXT NOT NULL,
                   age INTEGER,
                   phone TEXT,
                   created_at TEXT
);
''')

cursor.execute('''
CREATE TABLE IF NOT EXISTS honors(
                   id INTEGER PRIMARY KEY AUTOINCREMENT,
                   person_id INTEGER,
                   honor_name TEXT NOT NULL,
                   category TEXT,
                   issue_date TEXT,
                   description TEXT,
                   created_at TEXT,
                   FOREIGN KEY(person_id) REFERENCES persons(id)
);
''')
conn.commit()

# 侧边栏导航
menu = st.sidebar.selectbox("菜单", ["首页", "个人信息", "荣誉管理", "数据查询"])

if menu == "首页":
    st.header("系统概览")

    col1, col2 = st.columns(2)
    with col1:
        cursor.execute("SELECT COUNT(*) FROM persons;")
        st.metric("总人数", cursor.fetchone()[0])
    with col2:
        cursor.execute("SELECT COUNT(*) FROM honors;")
        st.metric("荣誉数量", cursor.fetchone()[0])

elif menu == "个人信息":
    st.header("👤 个人信息管理")

    # 添加个人信息
    with st.form("add_person"):
        st.subheader("添加个人信息")
        name = st.text_input("姓名*")
        age = st.number_input("年龄", min_value=0, max_value=150)
        phone = st.text_input("电话")

        if st.form_submit_button("保存"):
            if name:
                sql = f"INSERT INTO persons(name, age, phone, created_at) VALUES('{name}', {age}, '{phone}', datetime('now'));"
                cursor.execute(sql)
                conn.commit()
                st.success("添加成功!")

    # 显示和删除个人信息
    st.subheader("个人信息列表")
    cursor.execute("SELECT * FROM persons;")
    persons = cursor.fetchall()

    if persons:
        df = pd.DataFrame(persons, columns=['ID', '姓名', '年龄', '电话', '创建时间'])
        st.dataframe(df)

        # 删除功能
        person_ids = [f"{row[1]} (ID:{row[0]})" for row in persons]
        to_delete = st.selectbox("选择要删除的人员", [""] + person_ids)

        if to_delete and st.button("删除"):
            person_id = to_delete.split("ID:")[1].replace(")", "")
            cursor.execute(f"DELETE FROM honors WHERE person_id = {person_id};")
            cursor.execute(f"DELETE FROM persons WHERE id = {person_id};")
            conn.commit()
            st.success("删除成功!")
            st.rerun()

elif menu == "荣誉管理":
    st.header("🏆 荣誉信息管理")

    # 添加荣誉信息
    with st.form("add_honor"):
        st.subheader("添加荣誉信息")

        cursor.execute("SELECT id, name FROM persons;")
        persons = cursor.fetchall()

        if persons:
            person_options = {name: id for id, name in persons}
            selected_person = st.selectbox("选择人员", list(person_options.keys()))

            honor_name = st.text_input("荣誉名称*")
            category = st.selectbox("类别", ["荣誉", "竞赛", "证书", "其他"])
            issue_date = st.date_input("获得日期")
            description = st.text_area("描述")

            if st.form_submit_button("保存"):
                if honor_name:
                    sql = f"""
                    INSERT INTO honors(person_id, honor_name, category, issue_date, description, created_at)
                    VALUES({person_options[selected_person]}, '{honor_name}', '{category}', 
                           '{issue_date}', '{description}', datetime('now'));
                    """
                    cursor.execute(sql)
                    conn.commit()
                    st.success("荣誉添加成功!")
        else:
            st.warning("请先添加个人信息")

    # 显示和删除荣誉信息
    st.subheader("荣誉信息列表")
    cursor.execute('''
    SELECT h.id, p.name, h.honor_name, h.category, h.issue_date, h.description
    FROM honors h JOIN persons p ON h.person_id = p.id;
    ''')
    honors = cursor.fetchall()

    if honors:
        df = pd.DataFrame(honors, columns=['ID', '姓名', '荣誉名称', '类别', '获得日期', '描述'])
        st.dataframe(df)

        # 删除功能
        honor_ids = [f"{row[2]} - {row[1]}" for row in honors]
        to_delete = st.selectbox("选择要删除的荣誉", [""] + honor_ids)

        if to_delete and st.button("删除荣誉"):
            honor_id = honors[honor_ids.index(to_delete)][0]
            cursor.execute(f"DELETE FROM honors WHERE id = {honor_id};")
            conn.commit()
            st.success("荣誉删除成功!")
            st.rerun()

elif menu == "数据查询":
    st.header("🔍 数据查询")

    query_type = st.selectbox("查询类型", [
        "所有人员信息",
        "所有荣誉信息",
        "人员荣誉统计",
        "按类别统计荣誉"
    ])

    if query_type == "所有人员信息":
        cursor.execute("SELECT * FROM persons;")
        data = cursor.fetchall()
        if data:
            df = pd.DataFrame(data, columns=['ID', '姓名', '年龄', '电话', '创建时间'])
            st.dataframe(df)

    elif query_type == "所有荣誉信息":
        cursor.execute('''
        SELECT p.name, h.honor_name, h.category, h.issue_date, h.description
        FROM honors h JOIN persons p ON h.person_id = p.id;
        ''')
        data = cursor.fetchall()
        if data:
            df = pd.DataFrame(data, columns=['姓名', '荣誉名称', '类别', '获得日期', '描述'])
            st.dataframe(df)

    elif query_type == "人员荣誉统计":
        cursor.execute('''
        SELECT p.name, COUNT(h.id) as honor_count
        FROM persons p LEFT JOIN honors h ON p.id = h.person_id
        GROUP BY p.id, p.name;
        ''')
        data = cursor.fetchall()
        if data:
            df = pd.DataFrame(data, columns=['姓名', '荣誉数量'])
            st.dataframe(df)
            st.bar_chart(df.set_index('姓名'))

    elif query_type == "按类别统计荣誉":
        cursor.execute('''
        SELECT category, COUNT(*) as count
        FROM honors
        GROUP BY category;
        ''')
        data = cursor.fetchall()
        if data:
            df = pd.DataFrame(data, columns=['类别', '数量'])
            st.dataframe(df)

# 关闭连接
conn.close()
st.markdown("---")
st.caption("个人信息管理系统 | 双表结构 | 完整SQL操作")
