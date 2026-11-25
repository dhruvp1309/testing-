import random
from django.core.cache import cache
from django.db import connection
from django.db.models import Q
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from .models import Student


def _get_student_course_info(student_id):
    clean_id = str(student_id or '').strip()
    if not clean_id:
        return {'certificate_url': '', 'course_id': ''}

    certificate_url = ''
    course_id = ''

    # Try fetching both CertificateUrl and CourseId from dbo.a_StudentCourse
    try:
        sql_with_course = """
            SELECT TOP 1
                CertificateUrl,
                CourseId
            FROM dbo.a_StudentCourse
            WHERE CAST(StudentId AS NVARCHAR(100)) = %s
            ORDER BY
                CASE WHEN CourseId IS NOT NULL AND LTRIM(RTRIM(CAST(CourseId AS NVARCHAR(100)))) <> '' THEN 0 ELSE 1 END,
                CASE WHEN CertificateUrl IS NOT NULL AND LTRIM(RTRIM(CAST(CertificateUrl AS NVARCHAR(MAX)))) <> '' THEN 0 ELSE 1 END
        """
        with connection.cursor() as cursor:
            cursor.execute(sql_with_course, [clean_id])
            row = cursor.fetchone()

        if row:
            certificate_url = str(row[0] or '').strip()
            course_id = str(row[1] or '').strip()
        return {'certificate_url': certificate_url, 'course_id': course_id}
    except Exception:
        # Fallback to your original CertificateUrl query if CourseId column is not present
        pass

    sql_cert_only = """
        SELECT TOP 1
            StudentId,
            CertificateUrl
        FROM dbo.a_StudentCourse
        WHERE CAST(StudentId AS NVARCHAR(100)) = %s
          AND CertificateUrl IS NOT NULL
          AND LTRIM(RTRIM(CAST(CertificateUrl AS NVARCHAR(MAX)))) <> ''
    """
    with connection.cursor() as cursor:
        cursor.execute(sql_cert_only, [clean_id])
        row = cursor.fetchone()

    if row:
        certificate_url = str(row[1] or '').strip()

    return {'certificate_url': certificate_url, 'course_id': ''}


def format_student_payload(student):
    clean_id = str(student.student_id).strip()
    clean_mobile = str(student.mobile).strip() if student.mobile else ''
    course_info = _get_student_course_info(clean_id)
    schedule_url = (
        f"http://lms.itcmogri.org:801/Admin/a_publicSec.aspx?StudentId={clean_id}&CourseId={course_info['course_id']}"
        if course_info['course_id']
        else ''
    )

    return {
        'student_id': clean_id,
        'name': student.full_name,
        'first_name': (student.first_name or '').strip(),
        'middle_name': (student.middle_name or '').strip(),
        'last_name': (student.last_name or '').strip(),
        'mobile': clean_mobile,
        'course_id': course_info['course_id'],
        'schedule_url': schedule_url,
        'certificate_url': course_info['certificate_url'],
    }


@api_view(['POST'])
def lookup_student(request):
    raw_id = request.data.get('student_id')
    if not raw_id:
        return Response({'error': 'Student ID is required'}, status=status.HTTP_400_BAD_REQUEST)

    str_id = str(raw_id).strip()
    query = Q(student_id__iexact=str_id) | Q(student_id=str_id)
    if str_id.isdigit():
        try:
            query |= Q(student_id=int(str_id))
        except ValueError:
            pass

    student = Student.objects.filter(query).first()
    if not student:
        return Response(
            {
                'error': 'Student ID not found',
                'requires_otp': True,
                'message': 'ID not found in direct records. Please verify with your Mobile Number.',
            },
            status=status.HTTP_404_NOT_FOUND,
        )

    return Response(format_student_payload(student), status=status.HTTP_200_OK)


@api_view(['POST'])
def send_otp(request):
    mobile = request.data.get('mobile')
    if not mobile:
        return Response({'error': 'Mobile number is required'}, status=status.HTTP_400_BAD_REQUEST)

    clean_mobile = str(mobile).strip()
    student = Student.objects.filter(mobile__icontains=clean_mobile).first()
    if not student:
        return Response({'error': 'Mobile number not found in database'}, status=status.HTTP_404_NOT_FOUND)

    otp = str(random.randint(100000, 999999))
    cache.set(f'otp_{clean_mobile}', otp, timeout=300)
    print(f'OTP for {clean_mobile} is: {otp}')

    return Response(
        {
            'message': 'OTP sent successfully',
            'mobile': clean_mobile,
            'debug_otp': otp,
        },
        status=status.HTTP_200_OK,
    )


@api_view(['POST'])
def verify_otp(request):
    mobile = request.data.get('mobile')
    otp = request.data.get('otp')

    if not mobile or not otp:
        return Response({'error': 'Mobile and OTP are required'}, status=status.HTTP_400_BAD_REQUEST)

    clean_mobile = str(mobile).strip()
    clean_otp = str(otp).strip()
    saved_otp = cache.get(f'otp_{clean_mobile}')

    if not saved_otp:
        return Response({'error': 'OTP expired or not found'}, status=status.HTTP_400_BAD_REQUEST)

    if str(saved_otp) != clean_otp:
        return Response({'error': 'Invalid OTP'}, status=status.HTTP_400_BAD_REQUEST)

    student = Student.objects.filter(mobile__icontains=clean_mobile).first()
    if not student:
        return Response({'error': 'Student profile not found'}, status=status.HTTP_404_NOT_FOUND)

    cache.delete(f'otp_{clean_mobile}')
    return Response(format_student_payload(student), status=status.HTTP_200_OK)
