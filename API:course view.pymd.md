```python
# coding: utf-8
"""
  @api {get} /v1.0/course/my/list get_my_courses
  @apiVersion 1.0.0
  @apiName get_my_courses
  @apiGroup course v1.0
  @apiDescription use to get my courses
  @apiSuccess (Success 200) {String} status 200
  @apiSuccess (Success 200) {String} message 操作成功
  @apiSuccess (Success 200) {List} content courses
"""
"""
  @api {get} /v1.0/course/types list_types
  @apiVersion 1.0.0
  @apiName list_types
  @apiGroup course v1.0
  @apiDescription to list all the course types
  @apiSuccess (Success 200) {String} status 200
  @apiSuccess (Success 200) {String} message 操作成功
  @apiSuccess (Success 200) {List} content courses_types
"""
"""
  @api {get} /v1.0/course/creators list_creator
  @apiVersion 1.0.0
  @apiName list_creator
  @apiGroup course v1.0
  @apiDescription to list all the course creators
  @apiSuccess (Success 200) {String} status 200
  @apiSuccess (Success 200) {String} message 操作成功
  @apiSuccess (Success 200) {List} content creators
"""
"""
  @api {get} /v1.0/course/type/list/with_detail with_detail
  @apiVersion 1.0.0
  @apiName with_detail
  @apiGroup course v1.0
  @apiDescription for students and admin to get all course types
  @apiSuccess (Success 200) {String} status 200
  @apiSuccess (Success 200) {String} message 操作成功
  @apiSuccess (Success 200) {List} content all course types
"""
"""
  @api {delete} /v1.0/course/type/<int:type_id>  delete_course_type
  @apiVersion 1.0.0
  @apiName delete_course_type
  @apiGroup course v1.0
  @apiDescription delete input course type
  @apiSuccess (Success 200) {String} status 200
  @apiSuccess (Success 200) {String} message 操作成功
  @apiErrorExample {json} 404错误
     HTTP/1.1 404 Not Found
     {
       "error": "类型不存在"
     }
  @apiErrorExample {json} 500错误
     HTTP/1.1 500 Internal Server Error
     {
       "error": "已存在所属课程，无法删除"
     }
"""
"""
 @api {post} /v1.0/course/type/<int:type_id> update_course_type
  @apiVersion 2.0.0
  @apiName update_course_type
  @apiGroup course v1.0
  @apiDescription update course type by type_id
  @apiParam {String} type_id 类型id
  @apiParam {String} data 'desc'
  @apiSuccess (Success 200) {String} status 200
  @apiSuccess (Success 200) {String} message 操作成功
  @apiErrorExample {json} 404错误
     HTTP/1.1 404 Not Found
     {
       "error": "类型不存在"
     }
"""
"""
 @api {post} /v1.0/course/type/create create_course_type
  @apiVersion 2.0.0
  @apiName create_course_type
  @apiGroup course v1.0
  @apiDescription create new course type
  @apiParam {String} newct 新的课程类型
  @apiSuccess (Success 200) {String} status 200
  @apiSuccess (Success 200) {String} message 操作成功
  @apiErrorExample {json} 500错误
     HTTP/1.1 500 Internal Server Error
     {
       "error": "类型名称已存在"
     }
"""
from flask import Blueprint, jsonify, request, escape, current_app, send_from_directory
from flask_login import login_required
from .utils import CourseHandler
import os
import uuid
from app.api.auth.utils import AuthHandler
from werkzeug.utils import secure_filename
import re
from app.api.utils.utils import FileHandler
from app.decorators import admin_required, activated_required


course = Blueprint('course', __name__, url_prefix='/v1.0/course')


@course.route('/type/list', methods=['GET'])
@login_required
@activated_required
@admin_required
def get_ctypes():
    return jsonify(CourseHandler.get_all_ctypes())


@course.route('/type/<int:type_id>/courses', methods=['GET'])
@login_required
@activated_required
@admin_required
def get_courses_by_type_id(type_id):
    return jsonify(CourseHandler.get_courses_by_type_id(type_id))


@course.route('/type/<int:type_id>', methods=['GET'])
@login_required
@activated_required
@admin_required
def get_ctype(type_id):
    return jsonify(CourseHandler.get_ctype(type_id))


@course.route('/resources', methods=['GET'])
@login_required
@activated_required
@admin_required
def get_new_course_resources():
    if not AuthHandler.validate_token():
        return jsonify({'status': 404, 'message': '云连接失败'})
    return jsonify(CourseHandler.get_new_course_resources())


@course.route('/create', methods=['POST'])
@login_required
@activated_required
@admin_required
def create_course():
    param = request.get_json(force=True, silent=False, cache=True)
    res = course_validator(param)
    if res['status']:
        return jsonify(CourseHandler.create_course(res['content']))
    else:
        return jsonify({'status': 500, 'message': res['message']})


@course.route('/upload/image', methods=['POST'])
@login_required
@activated_required
@admin_required
def upload_image():
    files_arr = request.files.getlist('images')
    res = []
    for rq_file in files_arr:

        file_str = rq_file.read()
        rv_file = FileHandler.file_validator_bin(file_str)
        if rv_file.lower() not in current_app.config['IMAGES']:
            return jsonify({'status': 500, 'message': u'格式错误'})

        filename = ''.join(str(uuid.uuid4()).split('-')) + '.' + rv_file.lower()
        path = os.path.join(current_app.config['IMAGE_UPLOAD_FOLDER'], filename)

        # rq_file.save(path)
        fh = open(path, 'w')
        fh.write(file_str)
        fh.close()

        res.append(filename)
    return jsonify({'status': 200, 'message': u'操作成功', 'content': res})


@course.route('/download/image/<string:filename>', methods=['GET'])
@login_required
@activated_required
@admin_required
def download_image(filename):
    folder = current_app.config['IMAGE_UPLOAD_FOLDER']
    secure_file = secure_filename(filename)
    if os.path.isfile(os.path.join(folder, secure_file)):
        return send_from_directory(folder, secure_file, as_attachment=False)
    return ''


@course.route('/upload/attachment', methods=["POST"])
@login_required
@activated_required
@admin_required
def upload_attachment():
    # handle one file at a time, to be consistent with FE
    rq_file = request.files.get('file')
    rq_file_names = rq_file.filename.split('.')
    if len(rq_file_names) <= 1 or (rq_file_names[-1] not in current_app.config['FILES']):
        return jsonify({'status': 500, 'message': u'格式错误'})
    file_str = rq_file.read()
    rv_file = FileHandler.file_validator_bin(file_str)
    if rv_file.lower() not in current_app.config['FILES'] and rv_file.lower() != "zip":
        return jsonify({'status': 500, 'message': u'格式错误'})

    filename = ''.join(str(uuid.uuid4()).split('-')) + '.' + rq_file_names[-1]
    path = os.path.join(current_app.config['FILE_UPLOAD_FOLDER'], filename)
    # rq_file.save(path)
    fh = open(path, 'w')
    fh.write(file_str)
    fh.close()

    return jsonify({'status': 200, 'message': u'操作成功',
                    'content': "http://{}/course/download/attachment/{}".format(current_app.config['FILE_SERVER'],
                                                                                filename)})


@course.route('/download/attachment/<string:filename>', methods=["GET"])
@login_required
@activated_required
@admin_required
def download_attachment(filename):
    folder = current_app.config['FILE_UPLOAD_FOLDER']
    secure_file = secure_filename(filename)
    print(os.path.join(folder, secure_file))
    if os.path.isfile(os.path.join(folder, secure_file)):
        return send_from_directory(folder, secure_file, as_attachment=False)
    return ''


@course.route('/detail/<int:course_id>', methods=['GET'])
@login_required
@activated_required
@admin_required
def get_course(course_id):
    if not AuthHandler.validate_token():
        return jsonify({'status': 404, 'message': '云连接失败'})
    return jsonify(CourseHandler.get_course(course_id))


@course.route('/my/list', methods=['GET'])
@login_required
@activated_required
@admin_required
def get_my_courses():
    return jsonify(CourseHandler.get_my_courses())


@course.route('/my/detail/<int:course_id>', methods=['GET'])
@login_required
@activated_required
@admin_required
def get_my_course(course_id):
    if not AuthHandler.validate_token():
        return jsonify({'status': 404, 'message': '云连接失败'})
    return jsonify(CourseHandler.get_my_course(course_id))


@course.route('/update/<int:course_id>', methods=["POST"])
@login_required
@activated_required
@admin_required
def update_course(course_id):
    if not AuthHandler.validate_token():
        return jsonify({'status': 404, 'message': '云连接失败'})
    param = request.get_json(force=True, silent=False, cache=True)
    res = course_validator(param)
    if res['status']:
        return jsonify(CourseHandler.update_course(course_id, res['content']))
    else:
        return jsonify({'status': 500, 'message': res['message']})


@course.route('/actions/<int:course_id>', methods=["POST"])
@login_required
@activated_required
@admin_required
def manipulate_course(course_id):
    params = request.get_json(force=True, silent=False, cache=True)
    allowed_actions = ['recall', 'delete', 'reclaim', 'submit']
    if params['action'] not in allowed_actions:
        return jsonify({"status": 400, "message": u'操作失败'})
    else:
        return jsonify(CourseHandler.modify_course_status(course_id, params['action']))


@course.route('/environment/create/<int:course_id>', methods=['GET'])
@login_required
@activated_required
@admin_required
def create_environment(course_id):
    if not AuthHandler.validate_token():
        return jsonify({'status': 404, 'message': '云连接失败'})
    return jsonify(CourseHandler.create_environment(course_id))


@course.route('/environment/<int:course_id>', methods=['DELETE'])
@login_required
@activated_required
@admin_required
def delete_environment(course_id):
    if not AuthHandler.validate_token():
        return jsonify({'status': 404, 'message': '云连接失败'})
    return jsonify(CourseHandler.delete_environment(course_id))


# todo
@course.route('/history/list', methods=['GET'])
@login_required
@activated_required
@admin_required
def get_history():
    return jsonify(CourseHandler.get_history_courses())


@course.route('/proposal/create/<int:course_id>', methods=['POST'])
@login_required
@activated_required
@admin_required
def create_proposal(course_id):
    params = request.get_json(force=True, silent=False, cache=True)
    res = proposal_validator(params)
    if not res['status']:
        return jsonify({'status': 500, 'message': res['message']})
    return jsonify(CourseHandler.create_proposal(course_id, res['content']))


# # deprecated
# @course.route('/proposal/update/<int:proposal_id>', methods=['POST'])
# @login_required
# @activated_required
# def update_proposal(proposal_id):
#     print(123)
#     params = request.get_json(force=True, silent=False, cache=True)
#     res = proposal_validator(params)
#     if not res['status']:
#         return jsonify({'status': 500, 'message': res['message']})
#     return jsonify(CourseHandler.update_proposal(proposal_id, res['content']))


@course.route('/proposal/list')
@login_required
@activated_required
@admin_required
def get_proposals():
    return jsonify(CourseHandler.get_proposals())


@course.route('/proposal/actions/<int:proposal_id>', methods=["POST"])
@login_required
@activated_required
@admin_required
def manipulate_proposal(proposal_id):
    params = request.get_json(force=True, silent=False, cache=True)
    allowed_actions = ["recall", "delete", "submit"]
    if params['action'] not in allowed_actions:
        return jsonify({"status": 400, "message": u'操作失败'})
    else:
        return jsonify(CourseHandler.modify_proposal_status(proposal_id, params['action']))


@course.route('/proposal/<int:proposal_id>', methods=["GET"])
@login_required
@activated_required
@admin_required
def get_proposal(proposal_id):
    return jsonify(CourseHandler.get_proposal(proposal_id))


@course.route('/type/list/with_detail', methods=['GET'])
@login_required
@activated_required
@admin_required
def get_ctypes_with_detail():
    return jsonify(CourseHandler.get_all_ctypes(detail=True))


@course.route('/type/<int:type_id>', methods=['POST'])
@login_required
@activated_required
@admin_required
def update_course_type(type_id):
    params = request.get_json(force=True, silent=False, cache=True)
    data = escape(params.get("desc", None))
    return jsonify(CourseHandler.update_course_type(type_id, data))


@course.route('/type/create', methods=['POST'])
@login_required
@activated_required
@admin_required
def create_course_type():
    params = request.get_json(force=True, silent=False, cache=True)
    data = params
    # todo validator
    return jsonify(CourseHandler.create_course_type(data))


@course.route('/type/<int:type_id>', methods=['DELETE'])
@login_required
@activated_required
@admin_required
def delete_course_type(type_id):
    return jsonify(CourseHandler.delete_course_type(type_id))


@course.route('/list', methods=['GET'])
@login_required
@activated_required
@admin_required
def list_course():
    # c_type = request.values.get("type", -1)
    # c_name = request.values.get("name", '')
    # c_status = request.values.get("status", -1)
    return jsonify(CourseHandler.list_course())


@course.route('/creators', methods=['GET'])
@login_required
@activated_required
@admin_required
def list_creator():
    return jsonify(CourseHandler.list_creators())


@course.route('/types', methods=['GET'])
@login_required
@activated_required
@admin_required
def list_types():
    return jsonify(CourseHandler.list_course_types())


@course.route('/<int:course_id>', methods=['DELETE'])
@login_required
@activated_required
@admin_required
def delete_course(course_id):
    return jsonify(CourseHandler.delete_course(course_id))


def proposal_validator(param):
    p_type = escape(param.get('p_type', ''))
    p_content = param.get('p_content', '')
    p_attachment = param.get('p_attachment', [])
    print(param)
    if not p_type and not (p_content or p_attachment):
        return {"status": False, "message": u"请填写必填项"}
    if p_type not in ['exploit', 'fix']:
        return {"status": False, "message": u"方案类型错误"}
    f_v = attachment_validator(p_attachment)
    if not f_v['status']:
        return f_v

    data = {'p_type': p_type, 'p_content': p_content, 'p_attachment': f_v['content']}
    return {"status": True, "message": u"操作成功", "content": data}


def course_validator(param):
    name = escape(param["c_name"])
    c_type = escape(param['c_type'])
    desc = escape(param['desc'])
    image = escape(param['image'])
    flavor = escape(param['flavor'])
    duration = escape(param['duration'])
    difficulty = escape(param['difficulty'])
    if not (name and c_type and desc and desc and image and flavor and duration and difficulty):
        return {"status": False, "message": u"请填写必填项"}

    content = param['content']
    attachment = param['attachment']
    if (not content) and (not attachment):
        return {"status": False, "message": u"请填写课程内容或上传课件"}

    http = escape(param['http'])
    udp = escape(param['udp'])
    ssh = escape(param['ssh'])
    if not ssh.isdigit() or int(ssh) not in [0, 1]:
        print(ssh, type(ssh))
        return {"status": False, "message": u"ssh参数值错误"}

    new_http = re.split(r'[\s,\;]+', http)if http else []
    new_udp = re.split(r'[\s,\;]+', udp)if udp else []
    http = ','.join(new_http)
    udp = ','.join(new_udp)
    ports = new_http + new_http
    for p in ports:
        if not p.isdigit() or int(p) not in range(0, 65536):
            print('端口无效')
            return {"status": False, "message": u"http/udp端口无效"}

    f_v = attachment_validator(attachment)
    if not f_v['status']:
        return f_v

    data = dict(
        name=name,
        c_type=c_type,
        desc=desc,
        content=content,
        attachment=f_v['content'],
        image=image,
        flavor=flavor,
        http=http,
        udp=udp,
        ssh=ssh,
        duration=duration,
        difficulty=difficulty
    )
    return {"status": True, "message": u"校验成功", "content": data}


def attachment_validator(attachment):
    res = []
    for a in attachment:
        res_item = dict()
        # for key in a:
        #     if key not in {'url', 'name'}:
        #         return {'status': False, 'message': u"参数名错误：" + key}
        res_item['url'] = escape(a['url']) if a.get('url', '') else ''
        res_item['name'] = escape(a['name']) if a.get('name', '') else ''
        secure_file = secure_filename(res_item.get('url', '').split('/')[-1])
        if res_item['url']:
            url_pattern = r'^http://192\.168\.1\.25:8888/course/download/attachment/[0-9a-z]{32}\.' \
                          r'\.(pptx|pdf|zip)$'
            if not re.match(url_pattern, res_item['url']):
                current_app.logger.error('wrong attachment url: {}'.format(res_item['url']))
                continue
        folder = current_app.config['FILE_UPLOAD_FOLDER']
        if not os.path.isfile(os.path.join(folder, secure_file)):
            print(os.path.join(folder, secure_file))
            return {"status": False, "message": u"文件不存在"}
        res.append(res_item)
    print(res)
    return {"status": True, "message": u"文件存在", "content":res}

```

