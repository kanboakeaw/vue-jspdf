<template>
  <div id="FormExample">
        <h1>แบบฟอร์ม วฐ. ของระบบวิทยฐานะ</h1>
      <div>

        <button @click="genaratePdf_B1">แบบฟอร์ม วฐ.บ.1</button>
        <button @click="genarateHTML">genarate pdf with HTML2Canvas</button>
        <button @click="genarateImage">HTML to image</button>
      </div>
      
  </div>
</template>

<script>
import jsPDF from "jspdf";
import html2canvas from 'html2canvas';
import HelloWorld from '@/components/HelloWorld.vue'

export default {
    name : 'PdfExample',
    components: {
      HelloWorld
    },
    data(){
        return{
            // for jsPDF : https://rawgit.com/MrRio/jsPDF/master/docs/jsPDF.html
            pdfOption: {
                orientation: "p",
                format: "a4",
                unit: "mm",
                lineHeight: 1,
                putOnlyUsedFonts : true
            },
            pdfConfig: {
                margin: {
                    t: 20,
                    r: 20,
                    b: 20,
                    l: 20,
                    line1Top: 30,
                },
                typo: {
                    header: 20,
                    large: 18,
                    normal: 16,
                    small: 8
                },
                lineHeight: {
                    header: 1.3,
                    large: 1.3,
                    normal: 1.3,
                    small: 1.3
                }
            },
            canvasOption: {
                scale: 2,
                useCORS: true,
                logging: false,
                backgroundColor: '#ffffff'
            }
        }
    },
    methods:{
        convertToThaiNumber(num) {
            const thaiDigits = ['๐', '๑', '๒', '๓', '๔', '๕', '๖', '๗', '๘', '๙'];
            return num.toString().split('').map(digit => thaiDigits[parseInt(digit)]).join('');
        },
        segmentTextByDictionary(text, locale = 'th') {
            const input = (text ?? '').toString();
            if (!input) return [];

            if (typeof Intl !== 'undefined' && typeof Intl.Segmenter === 'function') {
                const segmenter = new Intl.Segmenter(locale, { granularity: 'word' });
                return Array.from(segmenter.segment(input), ({ segment }) => segment);
            }

            return Array.from(input);
        },
        splitTextToSizeDictionary(pdf, text, maxWidth, locale = 'th') {
            const input = (text ?? '').toString();
            if (!input) return [''];

            const wrappedLines = [];
            const paragraphs = input.split(/\r?\n/);

            paragraphs.forEach((paragraph, index) => {
                if (paragraph === '') {
                    wrappedLines.push('');
                } else {
                    const tokens = this.segmentTextByDictionary(paragraph, locale);
                    let currentLine = '';

                    tokens.forEach(token => {
                        if (!token) return;

                        const candidate = `${currentLine}${token}`;
                        if (!currentLine || pdf.getTextWidth(candidate) <= maxWidth) {
                            currentLine = candidate;
                            return;
                        }

                        wrappedLines.push(currentLine);

                        const trimmedToken = token.replace(/^\s+/, '');
                        if (!trimmedToken) {
                            currentLine = '';
                            return;
                        }

                        if (pdf.getTextWidth(trimmedToken) <= maxWidth) {
                            currentLine = trimmedToken;
                            return;
                        }

                        const forcedSplit = pdf.splitTextToSize(trimmedToken, maxWidth);
                        if (!forcedSplit.length) {
                            currentLine = '';
                            return;
                        }

                        if (forcedSplit.length > 1) {
                            wrappedLines.push(...forcedSplit.slice(0, -1));
                        }
                        currentLine = forcedSplit[forcedSplit.length - 1];
                    });

                    if (currentLine) {
                        wrappedLines.push(currentLine);
                    }
                }

                if (index < paragraphs.length - 1 && paragraph !== '' && wrappedLines.length) {
                    // preserve explicit new line between paragraphs
                }
            });

            return wrappedLines.length ? wrappedLines : [''];
        },
        drawExpandedLine(pdf, text, x, y, targetWidth) {
            const chars = Array.from(text);
            if (chars.length <= 1) {
                pdf.text(text, x, y);
                return;
            }

            const widths = chars.map(ch => pdf.getTextWidth(ch));
            const textWidth = widths.reduce((a, b) => a + b, 0);
            const gap = targetWidth - textWidth;

            if (gap <= 0) {
                pdf.text(text, x, y);
                return;
            }

            const extra = gap / (chars.length - 1);
            let cursorX = x;

            chars.forEach((ch, i) => {
                pdf.text(ch, cursorX, y);
                cursorX += widths[i] + (i < chars.length - 1 ? extra : 0);
            });
        },
        drawParagraphWithIndent(pdf, text, x, y, maxWidth, indent = 0, lineHeightFactor = this.pdfConfig.lineHeight.normal, fontSize = this.pdfConfig.typo.normal, align = 'left') {
            const paragraphText = (text ?? '').toString().trim();
            if (!paragraphText) return y;

            pdf.setFontSize(fontSize);

            /**
             * Calculates the maximum width available for the first line of text.
             * Takes the larger value between 1 and the result of subtracting the indent
             * from the maximum width. This ensures a minimum width of 1 unit is always maintained.
             * @type {number}
             */
            const firstLineWidth = Math.max(1, maxWidth - indent);
            const firstLineSplit = this.splitTextToSizeDictionary(pdf, paragraphText, firstLineWidth, 'th');
            const firstLine = firstLineSplit[0] || '';

            const scaleFactor = pdf.internal.scaleFactor || 1;
            const lineHeight = (fontSize * lineHeightFactor) / scaleFactor;

            // วาดบรรทัดแรกแบบขยายเต็มความกว้าง
            this.drawExpandedLine(pdf, firstLine, x + indent, y, firstLineWidth);

            const remainingText = paragraphText.slice(firstLine.length).trim();
            if (!remainingText) return y + lineHeight;

            //วาดบรรทัดที่เหลือแบบปกติ โดยเริ่มที่ x เดิม (ไม่เยื้อง) และ y เพิ่มขึ้นตาม lineHeight
            let currentY = y + lineHeight;
            const remainingLines = this.splitTextToSizeDictionary(pdf, remainingText, maxWidth, 'th');

            remainingLines.forEach((line, i) => {
                const isLastLine = i === remainingLines.length - 1;
                if (isLastLine) {
                    pdf.text(line, x, currentY);
                } else {
                    this.drawExpandedLine(pdf, line, x, currentY, maxWidth);
                }
                currentY += lineHeight;
            });

            return currentY;
        },
        drawParagraphWithSubIndent(pdf, text, x, y, maxWidth, indent = 0, lineHeightFactor = this.pdfConfig.lineHeight.normal, fontSize = this.pdfConfig.typo.normal, align = 'left') {
            const paragraphText = (text ?? '').toString().trim();
            if (!paragraphText) return y;

            pdf.setFontSize(fontSize);

            const scaleFactor = pdf.internal.scaleFactor || 1;
            const lineHeight = (fontSize * lineHeightFactor) / scaleFactor;

            // แบ่ง segment ด้วย \n
            const segments = paragraphText.split('\n');
            let currentY = y;

            segments.forEach((segment) => {
            const trimmedSegment = segment.trim();
            if (!trimmedSegment) return;

            const firstLineSplit = this.splitTextToSizeDictionary(pdf, trimmedSegment, maxWidth, 'th');
            const firstLine = firstLineSplit[0] || '';
            const remainingText = trimmedSegment.slice(firstLine.length).trim();

            // ถ้าแต่ละ segment มีแค่บรรทัดเดียว ไม่ต้อง indent และไม่ต้องกระจายข้อความ
            if (!remainingText) {
                pdf.text(firstLine, x, currentY);
                currentY += lineHeight;
                return;
            }

            // ถ้า align === 'justify' ให้กระจายข้อความ มิฉะนั้นไม่กระจาย
            if (align === 'justify') {
                this.drawExpandedLine(pdf, firstLine, x, currentY, maxWidth);
            } else {
                pdf.text(firstLine, x, currentY);
            }
            currentY += lineHeight;

            const remainingLines = this.splitTextToSizeDictionary(pdf, remainingText, maxWidth - indent, 'th');
            remainingLines.forEach((line, i) => {
                const isLastLine = i === remainingLines.length - 1;
                if (isLastLine) {
                pdf.text(line, x + indent, currentY);
                } else {
                if (align === 'justify') {
                    this.drawExpandedLine(pdf, line, x + indent, currentY, maxWidth - indent);
                } else {
                    pdf.text(line, x + indent, currentY);
                }
                }
                currentY += lineHeight;
            });
            });

            return currentY;
        },
        calcLineHeightFromFontMetrics({
          fontSizePt,
          leadingRatio = 0.2,
          emHeightRatio = 0.85,
          scaleFactor = 1
        }) {
          const lineHeightPt = fontSizePt * (emHeightRatio + leadingRatio); // เช่น 16*(1+0.2)=19.2pt
          const lineHeightUnit = lineHeightPt / scaleFactor; // แปลงเป็นหน่วยเอกสาร (mm/pt/px)
          return { lineHeightPt, lineHeightUnit };
        },
        genaratePdf_B1() {
            try {
                let pdf = new jsPDF(this.pdfOption);
                const pdf_width = pdf.internal.pageSize.width;
                const pdf_height = pdf.internal.pageSize.height;
                const margin_l = this.pdfConfig.margin.l;
                const margin_r = this.pdfConfig.margin.r;

                let pdf_position_y = this.pdfConfig.margin.line1Top;

                pdf.setFont('THSarabunNew','normal');
                //pdf.setFontSize(this.pdfConfig.typo.header); 
                pdf.setTextColor('#000000');

                // เริ่มหน้าแรกของ วฐ.บ.1
                ////////////////////////

                pdf.text('วฐ.บ.๑', pdf_width - margin_r, pdf_position_y, { 
                // จัดวางข้อความ
                align: 'right',                 // 'left' | 'center' | 'right' | 'justify'
                baseline: 'alphabetic',         // 'alphabetic' | 'ideographic' | 'bottom' | 'top' | 'middle' | 'hanging'

                // การหมุน
                angle: 0,                       // องศาที่หมุน
                rotationDirection: 1,           // 0 = ทวนเข็ม, 1 = ตามเข็ม

                // ระยะและสัดส่วนตัวอักษร
                charSpace: 0,                   // ระยะห่างระหว่างตัวอักษร
                horizontalScale: 1,             // สเกลแนวนอนของตัวอักษร (1 = ปกติ)
                lineHeightFactor: 1.15,         // ตัวคูณระยะบรรทัด

                // การตัดบรรทัด
                maxWidth: 0,                    // >0 เพื่อบังคับตัดบรรทัดตามความกว้าง

                // โหมดการวาดข้อความ
                renderingMode: 'fill',          // 'fill' | 'stroke' | 'fillThenStroke' | 'invisible'
                                                // และโหมด clipping ที่ jsPDF รองรับ

                // encoding flags
                flags: {
                    autoencode: true,
                    noBOM: true
                }
                });

                //สร้างกรอบ สี่เหลี่ยม
                pdf.setLineWidth(0.1);
                /**
                 * doc.rect(x, y, w, h, style?)
                 *
                 * พารามิเตอร์:
                 * - x {number}      : พิกัดแกน X ของมุมซ้ายบน
                 * - y {number}      : พิกัดแกน Y ของมุมซ้ายบน
                 * - w {number}      : ความกว้าง
                 * - h {number}      : ความสูง
                 * - style {string|null} [optional]
                 *    'S'  = Stroke (วาดเฉพาะเส้นขอบ)          [ค่าเริ่มต้น]
                 *    'F'  = Fill (ระบายด้านใน)
                 *    'DF' = Fill + Stroke (ระบายและวาดขอบ)
                 *    'FD' = Fill + Stroke (เหมือน DF)
                 *    null = ยังไม่ paint ทันที (สะสม path ไว้ใช้ต่อ)
                 *
                 * หมายเหตุ:
                 * - หน่วยขึ้นกับตอนสร้างเอกสาร เช่น mm, pt, px
                 * - สีเส้นขอบใช้ setDrawColor(), ความหนาใช้ setLineWidth()
                 * - สีเติมใช้ setFillColor()
                 */
                pdf.rect(20, 53, 170, 110);

                //ข้อความในกรอบ
                pdf_position_y = 61;
                pdf.setFontSize(this.pdfConfig.typo.normal);

                pdf.text('แบบบันทึกการประเมินด้านที่ ๑ ', 23, pdf_position_y, { align: 'left' });
                let content = 'ด้านวินัย คุณธรรม จริยธรรม และจรรยาบรรณข้าราชการทหารที่ทำ หน้าที่สอน (สำหรับทุกวิทยฐานะ)';
                const contentLines = this.splitTextToSizeDictionary(pdf, content, 100, 'th');
                pdf.text(
                    contentLines,
                    86,
                    pdf_position_y,
                    { align: 'left', lineHeightFactor: this.pdfConfig.lineHeight.normal }
                );

                // Calculate text height properly with standard line height metrics
                const splitContent = this.splitTextToSizeDictionary(pdf, content, 100, 'th');
                const fontSize = this.pdfConfig.typo.normal;
                const { lineHeightUnit: contentLineHeight } = this.calcLineHeightFromFontMetrics({
                    fontSizePt: fontSize,
                    leadingRatio: 0.3,
                    emHeightRatio: 1,
                    scaleFactor: pdf.internal.scaleFactor
                });
                pdf.setFontSize(fontSize);
                pdf_position_y += splitContent.length * contentLineHeight;
                pdf.text('ประเมิน', 86, pdf_position_y, { align: 'left' });


                pdf_position_y += 1 * contentLineHeight;     
                // วาดวงกลม
                const circleX = 110;
                const circleRadius = 1.4;
                pdf.setDrawColor('#000000');
                pdf.setLineWidth(0.1);
                pdf.circle(circleX, pdf_position_y - circleRadius, circleRadius, 'S');
                
                // ข้อความพร้อมเว้นวรรณ
                pdf.text('ครั้งที่ ๑', 115, pdf_position_y, { align: 'left' });

                pdf_position_y += 1 * contentLineHeight;     
                // วาดวงกลม
                pdf.setDrawColor('#000000');
                pdf.setLineWidth(0.1);
                pdf.circle(circleX, pdf_position_y - circleRadius, circleRadius, 'S');
                
                // ข้อความพร้อมเว้นวรรณ
                pdf.text('ครั้งที่ ๒ (หลังจากพัฒนาครั้งที่ ๑)', 115, pdf_position_y, { align: 'left' });

                pdf_position_y += 1 * contentLineHeight;     
                // วาดวงกลม
                pdf.setDrawColor('#000000');
                pdf.setLineWidth(0.1);
                pdf.circle(circleX, pdf_position_y - circleRadius, circleRadius, 'S');
                
                // ข้อความพร้อมเว้นวรรณ
                pdf.text('ครั้งที่ ๓ (หลังจากพัฒนาครั้งที่ ๒)', 115, pdf_position_y, { align: 'left' });

                pdf_position_y += 10; 
                pdf.text('ข้อมูลผู้รับการประเมิน', 40, pdf_position_y, { align: 'left' });
                
                pdf_position_y += contentLineHeight;
                pdf.text('ชื่อ.................................................................................................................................................', 40, pdf_position_y, { align: 'left' });

                //พิมพ์ชื่อ และ นามสกุล โดยรับค่าจากตัวแปร 
                //ให้นำค่าจาก json มาแสดงในช่องชื่อ และนามสกุล 
                 const name = 'น.อ. สมชาย ';
                 const surname = 'ก้านบัวแก้ว';
                 const postfix = ' ร.น.'; // ให้ตรวจสอบว่าเป็นหทารเรือหรือไม่ ถ้าใช่ให้เติม ร.น. ต่อท้าย

                 pdf_position_y += -1.2;
                const fullName = `${name}${surname}${postfix}`;
                pdf.text(fullName, 80, pdf_position_y, { align: 'left' });



                pdf_position_y += contentLineHeight+1.2;
                pdf.text('วิทยฐานะปัจจุบัน..........................................................................................................................', 40, pdf_position_y, { align: 'left' });

                //พิมพ์วิทยฐานะปัจจุบัน โดยรับค่าจากตัวแปร
                //ให้นำค่าจาก json มาแสดง
                 const currentRank = 'ไม่มีวิทยฐานะ';

                pdf_position_y += -1.2;        
                pdf.text(currentRank, 80, pdf_position_y, { align: 'left' });

                pdf_position_y += contentLineHeight+1.2;
                pdf.text('สถานศึกษา/หน่วยงาน..................................................................................................................', 40, pdf_position_y, { align: 'left' });

                //พิมพ์สถานศึกษา/หน่วยงาน โดยรับค่าจากตัวแปร
                //ให้นำค่าจาก json มาแสดง
                 const institution = 'โรงเรียนช่างฝีมือทหาร';

                pdf_position_y += -1.2;        
                pdf.text(institution, 80, pdf_position_y, { align: 'left' });

                pdf_position_y += contentLineHeight+1.2;
                pdf.text('สังกัด.............................................................................................................................................', 40, pdf_position_y, { align: 'left' });
                //พิมพ์สถานศึกษา/หน่วยงาน โดยรับค่าจากตัวแปร
                //ให้นำค่าจาก json มาแสดง
                 const affiliation = 'สถาบันวิชาการป้องกันประเทศ';

                pdf_position_y += -1.2;        
                pdf.text(affiliation, 80, pdf_position_y, { align: 'left' });

                pdf_position_y += contentLineHeight+1.2;
                pdf.text('วิทยฐานะที่ขอรับการประเมิน........................................................................................................', 40, pdf_position_y, { align: 'left' });
               //พิมพ์สถานศึกษา/หน่วยงาน โดยรับค่าจากตัวแปร
                //ให้นำค่าจาก json มาแสดง
                const rankToBeEvaluated = 'ครูทหารชำนาญการพิเศษ';
                pdf_position_y += -1.2;        
                pdf.text(rankToBeEvaluated, 100, pdf_position_y, { align: 'left' });

                //เริ่มทำตอนที่ 1
                pdf.addPage();
                pdf_position_y = this.pdfConfig.margin.t;
                //pdf_position_y += this.pdfConfig.typo.header * this.pdfConfig.lineHeight.header;  
                pdf.text('ตอนที่ ๑  การมีวินัย (๒๐ คะแนน)', 40, pdf_position_y, { align: 'left' });

                // สร้างตาราง 3 คอลัมน์ พร้อม header ที่ repeat ในหน้าใหม่
                const tableData = [
                    {  col1: '๑.๑ การมีวินัยในตนเอง ยอมรับ และถือ ปฏิบัติตามกฎกติกามารยาทขนบธรรมเนียมและแบบแผนอันดีงามของสังคม (๔ คะแนน)', col2: '๑. รายการที่ 1\n๒. รายการที่ 2\n๓. รายการที่ 3\n๔. รายการที่ 4', col3: '๔  มีวินัยในตนเอง ยอมรับและถือปฏิบัติ    ตามกฎ กติกา มารยาท ขนบธรรมเนียม    และแบบแผนอันดีงามของสังคม เป็นแบบอย่างที่ดี และเป็นผู้นำ ในการเสริมสร้างพัฒนาผู้อื่นในด้านนี้\n๓  มีวินัยในตนเอง ยอมรับและถือปฏิบัติ ตามกฎ กติกา มารยาท ขนบธรรมเนียม  และแบบแผนอันดีงามของสังคม เป็นแบบอย่างที่ดี และมีส่วนร่วม  ในการเสริมสร้างพัฒนาผู้อื่นในด้านนี้\n๒  มีวินัยในตนเอง ยอมรับและถือปฏิบัติ ตามกฎ กติกา มารยาท ขนบธรรมเนียม และแบบแผนอันดีงามของสังคม เป็นแบบอย่างที่ดี \n๑  มีวินัยในตนเอง ยอมรับและถือปฏิบัติ  ตามกฎ กติกา มารยาท ขนบธรรมเนียม และแบบแผนอันดีงามของสังคม' },   
                    {  col1: '๑.๒ การรักษาและ เสริมสร้างวินัยในตำแหน่งหน้าที่ราชการ การปฏิบัติตามกฎหมาย ระเบียบแบบแผน ของทางราชการ (๔ คะแนน)', col2: '๔. รายงานผลการประเมินตนเองของ กองมาตรฐานการฝึกศึกษาทหาร กองบัญชาการ สถาบันวิชาการป้องกันประเทศ หลักสูตรข้าราชการทหารที่ทำหน้าที่สอนของกองบัญชาการกองทัพไทยรุ่นที่ ๕ ฉบับนี้ ใช้มาตรฐานและตัวบ่งชี้การประกันคุณภาพการฝึกอบรมทางทหาร บก.ทท. พ.ศ. ๒๕๖๗ เป็นแนวทาง ๕ ในการจัดทำรายงานการประเมินตนเอง การนำเสนอข้อมูลในรายงานการประเมินตนเอง ประกอบด้วย บทสรุปสำหรับผู้บริหาร ข้อมูลพื้นฐานของหน่วย ผลดำเนินงานประกันคุณภาพการฝึกอบรมทางทหาร บก.ทท. พ.ศ. ๒๕๖๗ (ตั้งแต่ ๑๕ ม.ค.-๒๘ มี.ค. ๖๗) การวิเคราะห์ภาพรวมระดับหลักสูตร ตลอดจนรายละเอียดข้อมูลเพิ่มเติม ในภาคผนวก 	กองมาตรฐานการฝึกศึกษาทหาร กองบัญชาการ สถาบันวิชาการป้องกันประเทศ หลักสูตรข้าราชการทหารที่ทำหน้าที่สอนของกองบัญชาการกองทัพไทย หวังเป็นอย่างยิ่งว่า รายงานการประเมินตนเองฉบับนี้ จะเป็นประโยชน์ต่อคณะทำงานประกันคุณภาพการฝึกอบรมทางทหาร บก.ทท. ที่จะช่วยให้ข้อเสนอแนะเพื่อการพัฒนา กองมาตรฐานการฝึกศึกษาทหาร กองบัญชาการ สถาบันวิชาการป้องกันประเทศ หลักสูตรข้าราชการทหารที่ทำหน้าที่สอนของกองบัญชาการกองทัพไทย ต่อไป \n๑. สถาบันวิชาการป้องกันประเทศ (สปท.) จัดตั้งขึ้นตาม พ.ร.ฎ. แบ่งส่วนราชการและกำหนดหน้าที่ของส่วนราชการ บก.ทหารสูงสุด กห. พ.ศ.๒๕๓๓ (ประกาศ ในราชกิจจานุเบกษา ฉบับพิเศษ เล่ม ๑๐๗ ตอนที่ ๑๖๕ หน้า ๔-๑๓ ลง ๕ กันยายน ๒๕๓๓) ให้ไว้ ณ วันที่ ๒๙ สิงหาคม พ.ศ.๒๕๓๓ (ถือเป็นวันสถาปนา) มาตรา ๔ (๑๘) มีหน้าที่พิจารณา เสนอความเห็น วางแผน อำนวยการ ประสานงาน กำกับดูแล และดำเนินการเกี่ยวกับการประศาสน์วิทยาการทางด้านความมั่นคงแห่งชาติ การป้องกันราชอาณาจักร การยุทธผสม การยุทธร่วม และการอำนวยการในระดับสูง การศึกษาอบรมเกี่ยวกับการสงคราม การเมืองและการปฏิบัติการจิตวิทยา รวมทั้งการวิจัยทางยุทธศาสตร์ และมาตรา ๔ (๑๙) กรมการศึกษา มีหน้าที่ พิจารณา เสนอความเห็น อำนวยการ ประสานงาน กำกับดูแล และดำเนินการ เกี่ยวกับการฝึกศึกษาทางทหาร การเผยแพร่วิทยาการเกี่ยวกับการทหาร การศึกษาและรวบรวมประวัติศาสตร์และโบราณคดีเกี่ยวกับการทหาร จัดและดำเนินการพิพิธภัณฑ์ทหารและห้องสมุดของกองบัญชาการทหารสูงสุด รวมทั้งดำเนินงานเกี่ยวกับกิจการของสภาการศึกษาวิชาการทหาร พ.ร.ฎ. แบ่งส่วนราชการ และกำหนดหน้าที่ของส่วนราชการ บก.ทหารสูงสุด กห. (ฉบับที่ ๒) พ.ศ. ๒๕๓๕ (ประกาศ ในราชกิจจานุเบกษา เล่ม ๑๐๙ ตอนที่ ๑๒๕ หน้า ๓๕-๓๘ ลง ๒๙ ธันวาคม ๒๕๓๕) มาตรา ๔ (๑๘) สถาบันวิชาการป้องกันประเทศ มีหน้าที่พิจารณา เสนอความเห็น วางแผน อำนวยการ ประสานงาน กำกับดูแล และดำเนินการเกี่ยวกับการศึกษาในระดับสูง การประศาสน์วิทยาการทางด้านความมั่นคงแห่งชาติ การป้องกันราชอาณาจักร การยุทธผสม การยุทธร่วมและการอำนวยการในระดับสูง \n๒. การศึกษาอบรมเกี่ยวกับสงครามการเมืองและการปฏิบัติการจิตวิทยา รวมทั้งการวิจัยทางยุทธศาสตร์ และมาตรา ๔ (๑๙) กรมยุทธศึกษาทหาร มีหน้าที่ พิจารณา เสนอความเห็น วางแผน อำนวยการ ประสานงาน กำกับดูแล และดำเนินการเกี่ยวกับการศึกษาทางทหาร การเผยแพร่วิทยาการเกี่ยวกับการทหาร การศึกษาและรวบรวมประวัติศาสตร์และโบราณคดีเกี่ยวกับการทหาร จัดและดำเนินการ พิพิธภัณฑ์ทหาร และห้องสมุดของกองบัญชาการทหารสูงสุด รวมทั้งดำเนินงานเกี่ยวกับกิจการสภาการศึกษาวิชาการทหาร ', col3: '๔  ตรงต่อเวลา ปฏิบัติงานตามที่ได้รับ มอบหมายได้สำเร็จและอุทิศเวลาอย่างต่อเนื่อง \n๓   ตรงต่อเวลาและปฏิบัติงานตามที่ได้รับ มอบหมายได้สำเร็จและอุทิศเวลา \n๒   ตรงต่อเวลาและปฏิบัติงานตามที่ได้รับมอบหมายได้สำเร็จ \n๑   ปฏิบัติงานตามปกติ' },
                    {  col1: '๑.๓ การตรงต่อเวลา การอุทิศเวลา ให้แก่ทางราชการ และผู้เข้ารับการสอน ฝึก ศึกษาอย่างต่อเนื่อง (๔ คะแนน)', col2: '๑. รายละเอียด', col3: 'หมายเหตุ' },
                ];

                const col1Width = 40;
                const col2Width = 70;
                const col3Width = 70;
                const rowHeight = 8;
                const headerHeight = 25;
                const pageHeight = pdf.internal.pageSize.height;
                const bottomMargin = 20;
                const maxContentHeight = pageHeight - pdf_position_y - bottomMargin;

                // เพิ่มระยะก่อนวาดตาราง
                pdf_position_y += 2;

                // วาด header ตาราง
                const drawTableHeader = (startY) => {
                    pdf.setDrawColor(0, 0, 0);
                    pdf.setLineWidth(0.1);
                    pdf.setFontSize(16);
                    pdf.rect(margin_l, startY, col1Width, headerHeight, 'S');
                    pdf.rect(margin_l + col1Width, startY, col2Width, headerHeight, 'S');
                    pdf.rect(margin_l + col1Width + col2Width, startY, col3Width, headerHeight, 'S');
                    
                    pdf.setFontSize(this.pdfConfig.typo.normal);
                    pdf.text('ตัวบ่งชี้', margin_l + 15, startY + 14);

                
                let hcolumn2 = 'บันทึกร่องรอยพฤติกรรมที่แสดงลักษณะ ตัวบ่งชี้ (ระบุข้อมูล/สารสนเทศ/เอกสาร หลักฐานที่สะท้อนพฤติกรรม)';
                let maxcol2Width = 62;
                const hcolumn2Lines = this.splitTextToSizeDictionary(pdf, hcolumn2, maxcol2Width, 'th');
                pdf.text(
                    hcolumn2Lines,
                    95,
                    pdf_position_y+7,
                    { maxWidth: maxcol2Width, align: 'center', lineHeightFactor: this.pdfConfig.lineHeight.normal }
                );

                pdf_position_y += 7;  
                let maxcol3Width = 62; 
                pdf.text(
                    'ผลการประเมิน',
                    165,
                    pdf_position_y,
                    { maxWidth: maxcol3Width, align: 'center', lineHeightFactor: this.pdfConfig.lineHeight.normal }
                );
                let hcolumn3 = 'ให้วงกลมล้อมรอบตัวเลข ตามพฤติกรรม ที่สอดคล้องกับหลักฐานร่องรอยที่บันทึกไว้';
                
                const hcolumn3Lines = this.splitTextToSizeDictionary(pdf, hcolumn3, maxcol3Width, 'th');
                pdf.text(
                    hcolumn3Lines,
                    165,
                    pdf_position_y+7,
                    { maxWidth: maxcol3Width, align: 'center', lineHeightFactor: this.pdfConfig.lineHeight.normal }
                );

                };

                // วาด body ตาราง
                drawTableHeader(pdf_position_y);
                
               // pdf_position_y += 10;
                pdf_position_y += headerHeight-7; // ปรับระยะหลัง header ให้อยู่ใกล้กับเนื้อหามากขึ้น

                let currentRowHeight = 0;
                pdf.setLineWidth(0.1);

                let maxWidth =0;
                let subindent = 5; // mm
                let paragraphText = ''; 

                const availableHeight = pageHeight - this.pdfConfig.margin.t - bottomMargin;
                const { lineHeightUnit: rowlineHeightUnit } = this.calcLineHeightFromFontMetrics({
                    fontSizePt: this.pdfConfig.typo.normal,
                    leadingRatio: 0.3,
                    emHeightRatio: 1,
                    scaleFactor: pdf.internal.scaleFactor
                });
                const maxLinesPerPage = Math.floor(availableHeight / rowlineHeightUnit)-4; // ลบ 4 บรรทัดเผื่อความไม่แน่นอนของการคำนวณ
                // วนลูปวาดแต่ละแถว แล้วคำนวนความสูงของแถวเพื่อเช็คว่าควรขึ้นหน้าใหม่หรือไม่
                // หากความยาวเกิน สร้างข้อมูลใหม่ โดยการเแยก ข้อความ และเพิ่ม row ใหม่เข้าไปใน tableData เพื่อให้แสดงผลได้ครบถ้วน
                let tableDataNew = [];
                tableData.forEach((row, index) => {
                     //const col1Lines = this.splitTextToSizeDictionary(pdf, row.col1, 30, 'th');
                     const col2Lines = this.splitTextToSizeDictionary(pdf, row.col2, col2Width - 4, 'th');
                     //const col3Lines = this.splitTextToSizeDictionary(pdf, row.col3, col3Width - 4, 'th');
                     //const countLines = Math.max(col1Lines.length, col2Lines.length, col3Lines.length);
                     const countLines = col2Lines.length;
                    let maxGroupText = Math.floor(countLines / maxLinesPerPage) + 1;
                    
                    if(countLines > maxLinesPerPage) {
                        for(let i=0; i<maxGroupText; i++) {
                            if(i===0){
                                tableDataNew.push({
                                    col1: row.col1,
                                    col2: col2Lines.slice(i*maxLinesPerPage, (i+1)*maxLinesPerPage).join('\n'),
                                    col3: row.col3
                                });
                            }
                            else {
                                tableDataNew.push({
                                    col1: ' ',
                                    col2: col2Lines.slice(i*maxLinesPerPage, (i+1)*maxLinesPerPage).join('\n'),
                                    col3: ' '
                                });
                            }
                            // tableDataNew.push({
                            //     col1: col1Lines.slice(i*maxLinesPerPage, (i+1)*maxLinesPerPage).join(''),
                            //     col2: col2Lines.slice(i*maxLinesPerPage, (i+1)*maxLinesPerPage).join('\n'),
                            //     col3: col3Lines.slice(i*maxLinesPerPage, (i+1)*maxLinesPerPage).join('')
                            // });
                        }
                    }
                    else {
                        tableDataNew.push({
                            col1: row.col1,
                            col2: row.col2,
                            col3: row.col3
                        });
                    }
                });


                tableDataNew.forEach((row, index) => {
                    // คำนวณความสูงที่ต้องการสำหรับแถวปัจจุบัน
                    const estimatedRowHeight = (Math.max(
                        this.splitTextToSizeDictionary(pdf, row.col1, col1Width - 4, 'th').length,
                        this.splitTextToSizeDictionary(pdf, row.col2, col2Width - 4, 'th').length,
                        this.splitTextToSizeDictionary(pdf, row.col3, col3Width - 4, 'th').length
                    ) * this.calcLineHeightFromFontMetrics({
                        fontSizePt: this.pdfConfig.typo.normal,
                        leadingRatio: 0.3,
                        emHeightRatio: 1,
                        scaleFactor: pdf.internal.scaleFactor
                    }).lineHeightUnit);

                    // ตรวจสอบว่าแถวปัจจุบันจะขึ้นหน้าใหม่หรือไม่
                    if (pdf_position_y + estimatedRowHeight > pageHeight - bottomMargin) {
                        pdf.addPage();
                        pdf_position_y = this.pdfConfig.margin.t;
                        drawTableHeader(pdf_position_y);
                        pdf_position_y += headerHeight - 7;
                    }

                    pdf.setFontSize(this.pdfConfig.typo.normal);
                    //คอลัมน์แรกให้พิมพ์ตัวเลขแบบไทย และข้อความในคอลัมน์ที่ 1 ให้พิมพ์แบบขยายเต็มความกว้างของคอลัมน์
                    //pdf.text(row.col_num, margin_l + 2, pdf_position_y + 5);

                    maxWidth = col1Width; // กำหนดความกว้างสูงสุดสำหรับคอลัมน์ที่ 1 โดยหักระยะขอบเล็กน้อย
                    subindent =7 // mm
                    paragraphText = (row.col1 ?? '').toString();

                    if (paragraphText.trim()) {
                         this.drawParagraphWithSubIndent(
                            pdf,
                            paragraphText,
                            margin_l,
                            pdf_position_y+5,
                            maxWidth - 4,
                            subindent,
                            this.pdfConfig.lineHeight.normal,
                            this.pdfConfig.typo.normal,
                            'left'
                        );
                    }

                    maxWidth = col2Width; // กำหนดความกว้างสูงสุดสำหรับคอลัมน์ที่ 1 โดยหักระยะขอบเล็กน้อย
                    subindent = 5; // mm
                    paragraphText = (row.col2 ?? '').toString();
                    if (paragraphText.trim()) {
                        this.drawParagraphWithSubIndent(
                            pdf,
                            paragraphText,
                            margin_l + col1Width + 2,
                            pdf_position_y+5,
                            maxWidth - 4,
                            subindent,
                            this.pdfConfig.lineHeight.normal,
                            this.pdfConfig.typo.normal,
                            'justify'
                        );
                    }

                    maxWidth = col3Width; // กำหนดความกว้างสูงสุดสำหรับคอลัมน์ที่ 1 โดยหักระยะขอบเล็กน้อย
                    subindent = 6; // mm
                    paragraphText = (row.col3 ?? '').toString();
                    if (paragraphText.trim()) {
                        this.drawParagraphWithSubIndent(
                            pdf,
                            paragraphText,
                            margin_l + col1Width + col2Width + 2,
                            pdf_position_y+5,
                            maxWidth - 4,
                            subindent,
                            this.pdfConfig.lineHeight.normal,
                            this.pdfConfig.typo.normal,
                            'justify'
                        );
                    }
                
                
                //     let contentLines = this.splitTextToSizeDictionary(pdf, row.col1, 30, 'th');
                //     pdf.text(
                //         contentLines,
                //         margin_l + 2,
                //         pdf_position_y+5,
                //         { align: 'left', lineHeightFactor: this.pdfConfig.lineHeight.normal }
                //     );

                //     contentLines = this.splitTextToSizeDictionary(pdf, row.col2, 65, 'th');
                //     pdf.text(
                //         contentLines,
                //         margin_l + col1Width + 2,
                //         pdf_position_y+5,
                //         { align: 'left', lineHeightFactor: this.pdfConfig.lineHeight.normal }
                //     );

                //    contentLines = this.splitTextToSizeDictionary(pdf, row.col3, 65, 'th');
                //     pdf.text(
                //         contentLines,
                //         margin_l + col1Width + col2Width + 2,
                //         pdf_position_y+5,
                //         { align: 'left', lineHeightFactor: this.pdfConfig.lineHeight.normal }
                //     );



                    //pdf.text(row.col1, margin_l + 2, pdf_position_y + 5);

                   // pdf.text(row.col2, margin_l + col1Width + 2, pdf_position_y + 5);
                   // pdf.text(row.col3, margin_l + col1Width + col2Width + 2, pdf_position_y + 5);

                    const col1Lines = this.splitTextToSizeDictionary(pdf, row.col1, 30, 'th');
                    const col2Lines = this.splitTextToSizeDictionary(pdf, row.col2, col2Width - 4, 'th');
                    const col3Lines = this.splitTextToSizeDictionary(pdf, row.col3, col3Width - 4, 'th');

                    const maxLines = Math.max(col1Lines.length, col2Lines.length, col3Lines.length);
                
                    currentRowHeight = (maxLines * this.calcLineHeightFromFontMetrics({
                        fontSizePt: this.pdfConfig.typo.normal,
                        leadingRatio: 0.3,
                        emHeightRatio: 1,
                        scaleFactor: pdf.internal.scaleFactor
                    }).lineHeightUnit); // เพิ่ม padding เล็กน้อย
                    


                    // วาดเส้นตารางสำหรับแต่ละแถว โดยใช้ความสูงที่คำนวณจากจำนวนบรรทัดในแต่ละคอลัมน์
                    pdf.rect(margin_l, pdf_position_y, col1Width, currentRowHeight, 'S');
                    pdf.rect(margin_l + col1Width, pdf_position_y, col2Width, currentRowHeight, 'S');
                    pdf.rect(margin_l + col1Width + col2Width, pdf_position_y, col3Width, currentRowHeight, 'S');

                    pdf_position_y += currentRowHeight;
                    pdf.setFontSize(this.pdfConfig.typo.normal);
                    
                    //คอลัมน์ที่ 1 ให้พิมพ์ตัวเลขแบบไทย และข้อความในคอลัมน์ที่ 1 ให้พิมพ์แบบขยายเต็มความกว้างของคอลัมน์
                    //pdf.text(row.col_num, margin_l + 5, pdf_position_y + 4);
                   // pdf.text(col1Lines, margin_l + 2, pdf_position_y + 4, { maxWidth: col1Width - 4, lineHeightFactor: 1 });
                    
                    //คอลัมน์ที่ 2
                    //pdf.text(col2Lines, margin_l + col1Width + 2, pdf_position_y + 4, { maxWidth: col2Width - 4, lineHeightFactor: 1 });

                    //คอลัมน์ที่ 3
                   // pdf.text(col3Lines, margin_l + col1Width + col2Width + 2, pdf_position_y + 4, { maxWidth: col3Width - 4, lineHeightFactor: 1 });

                    // pdf.rect(margin_l, pdf_position_y, col1Width, rowHeight);
                    // pdf.rect(margin_l + col1Width, pdf_position_y, col2Width, rowHeight);
                    // pdf.rect(margin_l + col1Width + col2Width, pdf_position_y, col3Width, rowHeight);
                    
                    //pdf_position_y += rowHeight; // ปรับระยะหลังวาดตารางให้อยู่ใกล้กับเนื้อหามากขึ้น
                });


















                pdf.addPage();
                pdf_position_y = this.pdfConfig.margin.t;            
                pdf.setFont('THSarabunNew','normal');
                pdf.setTextColor('#000000');
                pdf.setFontSize(this.pdfConfig.typo.normal);
                 //****************************เก็บไว้ใช้ได้ */               
                // const longText = '๒.๑ บก.ทท. (สส.ทหาร) จ้างพัฒนาระบบ ที่ Learning Management System และ Online Course ตามโครงการพัฒนาโครงสร้างพื้นฐานด้านเทคโนโลยีสารสนเทศ ประจำปีงบประมาณ พ.ศ. ๒๕๖๗ จำนวน ๑ งาน กับ บริษัท เพอร์เฟคคอมพิวเตอร์โซลูชัน จำกัด เป็นจำนวนเงินรวมทั้งสิ้น ๑๖,๑๔๘,๘๐๐.- บาท (สิบหกล้าน หนึ่งแสนสี่หมื่นแปดพันแปดร้อยบาทถ้วน) สัญญาจ้างฯ เลขที่ ๕๒/๒๕๖๗ ลง ๒๓ ส.ค. ๖๗ โดย สส.ทหาร รับไว้ใช้ในราชการ เมื่อ ๒ ก.ค. ๖๘ พร้อมการรับประกันความเสียหาย เป็นระยะเวลา ๓ ปี เริ่ม ๒ ก.ค. ๖๘ ครบกำหนด ๑ ก.ค. ๗๑ หากมีความชำรุดบกพร่องของพัสดุให้หน่วยผู้ใช้งานโดยตรงแจ้ง กจห.สส.ทหาร เพื่อประสาน บริษัท พอร์เฟคคอมพิวเตอร์โซลูชัน จำกัด ให้เร่งดำเนินการแก้ไขให้สามารถใช้ในราชการได้ตามที่สัญญาฯ กำหนด ก่อนสิ้นสุดระยะเวลารับประกัน ๓๐ วัน';
                // const maxWidth = pdf_width - margin_l - margin_r;
                // const indent = 25; // mm
                // const paragraphText = (longText ?? '').toString();

                // if (paragraphText.trim()) {
                //     pdf_position_y = this.drawParagraphWithIndent(
                //         pdf,
                //         paragraphText,
                //         margin_l,
                //         pdf_position_y,
                //         maxWidth,
                //         indent,
                //         this.pdfConfig.lineHeight.normal,
                //         this.pdfConfig.typo.normal,
                //         'justify'
                //     );
                // }
               //*****************************เก็บไว้ใช้ได้ */            

                // pdf.setTextColor('red');
                // pdf.setFontSize(this.pdfConfig.typo.small);
                
                // const longText2 = 'ทดสอบการสร้าง pdf โดย jsPdfddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddfdfdsf dfdsfsadf dsfsdfsadfasd dfsdfasdfsad fdsfsdfsdaf dddddddddddddddddddddddddddddd';
                // const paragraphText2 = (longText2 ?? '').toString();
                // if (paragraphText2.trim()) {
                //     const fontSize = this.pdfConfig.typo.small;
                //     const lineHeightFactor = this.pdfConfig.lineHeight.small;
                //     const firstLineWidth = Math.max(1, maxWidth - indent);
                //     const firstLineSplit = pdf.splitTextToSize(paragraphText2, firstLineWidth);
                //     const firstLine = firstLineSplit[0] || '';
                //     const scaleFactor = pdf.internal.scaleFactor || 1;
                //     const lineHeight = (fontSize * lineHeightFactor) / scaleFactor;

                //     pdf.text(firstLine, margin_l + indent, pdf_position_y, { maxWidth, align: 'justify', lineHeightFactor });

                //     const remainingText = paragraphText2.slice(firstLine.length).trimStart();
                //     if (remainingText) {
                //         pdf.text(remainingText, margin_l, pdf_position_y + lineHeight, { maxWidth, align: 'justify', lineHeightFactor });
                //         const remainingLines = pdf.splitTextToSize(remainingText, maxWidth);
                //         pdf_position_y += lineHeight + (remainingLines.length * lineHeight);
                //     } else {
                //         pdf_position_y += lineHeight;
                //     }
                // }


                pdf.addPage();
                pdf_position_y = this.pdfConfig.margin.t;
                pdf.setFont('Sarabun','normal');
                pdf.setFontSize(this.pdfConfig.typo.header);
                pdf.setTextColor('#025955');
                pdf.text('ทดสอบการสร้าง pdf โดย jsPdf', pdf_width / 2, pdf_position_y, null, null, "center");

                setTimeout(() => {
                    pdf.setFont('Sarabun','normal');
                    pdf.setFontSize(this.pdfConfig.typo.small);
                    pdf.setTextColor('#000');
                    const textDate = (new Date()).toString();
                    
                    const pages = pdf.internal.getNumberOfPages();

                    for (let j = 1; j < pages + 1 ; j++) {
                        pdf.setPage(j);
                        pdf.text(`วันเวลา : ${textDate}`, margin_l, pdf_height - 10);
                        const thaiNumber = this.convertToThaiNumber(j) + ' จาก ' + this.convertToThaiNumber(pages);
                        pdf.text(`หน้า ${thaiNumber}`, pdf_width - margin_r, pdf_height - 10, null, null, "right");
                    }

                    //pdf.save(Date.now() + ".pdf");
                    pdf.output('dataurlnewwindow');

                }, 0);
               

            }
            catch (err) {
                console.log(err);
            }
        }       
    }
}
</script>

<style>

</style>